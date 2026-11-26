# Changelog

## Unreleased

- Watcher budget: charge every rotate attempt (success or not) to the
  hourly/daily limit and advance the min-interval clock on the last
  attempt (`last_attempt`) instead of only on a successful rotate. The
  old behaviour charged only successes, so an exit that stayed burned
  retried on every timer tick without ever spending the budget. Guard
  the min-interval gate against a future `last_rotate`/`hour_start`/
  `last_attempt` (NTP step back) so it cannot freeze forever.

## v0.7.0 (2026-11-24)

- New egress method `warp`: each rotate registers a FRESH anonymous WARP device with wgcf (`wgcf register` + `wgcf generate`, random keypair + install_id, no personal account) and swaps the netns WireGuard peer to its profile, exactly like the Proton peer swap. Cloudflare assigns each new device a fresh IPv6 address; the rotate verifies the egress IPv6 /64 actually changed (via `WARP_EXIT_IP6_URL`, default api64.ipify.org) and retries with another fresh registration when the new egress /64 equals the previous one, up to 3 fresh registrations per rotate. The public IPv4 egress is Cloudflare's shared pool and may already be 429. `rotate --json` for warp also reports `exit_ip6`, `prefix64` (egress /64) and `profile_prefix64`. wgcf 2.3.1 is downloaded once (pinned release, SHA-256 verified); only the generated profile is kept (0600), the disposable account file is deleted right after generate. `doctor` gains warp checks (wgcf, profile, handshake, inside-ns egress, IPv6 exit). The old warp-cli daemon is NOT used by this method. LIVE RESULT 23 Nov: fresh egress v6 does NOT by itself give a 200 (warp rotate #1, egress v6 2a09:bac5:55fa:1d0f::2e5:65, gateway probe 429 while the same minute's Proton probe was 200; exit v4 was the already-burned shared 104.28.254.46). The gateway's upstream connection apparently goes over IPv4, or the v6 path is not what the upstream counts, so warp is NOT a working 429 escape as implemented. See GOAL.md for the full evidence and what would need proving before any live use.
- Correction: the "WARP tested dead" note was wrong. Checked live 23 Nov, the warp-cli daemon is Connected and its SOCKS proxy answers with a Cloudflare exit IP; what was pre-burned was that shared egress IP for the OpenCode free tier, not the tunnel. The README says so now. Note also that openhop proper does NOT register a new account per rotate: it keeps one registration and runs `warp-cli tunnel rotate-keys` for a fresh /64. FreeHop uses fresh wgcf registrations instead because warp-cli here runs in MASQUE proxy mode on host loopback, unreachable from the netns without a forwarder, while a wgcf profile plugs straight into the netns WireGuard interface.
- Hardening for switching methods: `rotate_proton` now re-asserts the interface private key from the Proton key file, drops any WARP profile addresses, restores `WG_ADDR`, and removes the IPv6 default route a warp rotate added, so a proton rotate after a warp rotate cannot leave the wrong key or a v6 leak into the Proton tunnel. All steps are idempotent when already on proton.
- Fix (found live 23 Nov, warp rotate #2): the warp verification compared the live egress /64 against the profile's /64 and required them equal. That always fails: Cloudflare NATs WARP egress to 2a09:bac5::/32 while the profile address sits in 2606:4700:110::/48. Rotate #1 only passed because there was no previous warp state to compare against; #2 failed all 3 fresh registrations on this check. The check now compares the live egress /64 against the PREVIOUS egress /64 (a rotate only counts when the egress identity actually changes), and the state records both: `prefix64` is the egress /64, `profile_prefix64` is the registration's own /64. `rotate --json` for warp reports both.
- Fix (found live 23 Nov, same incident): after the failed warp rotate #2, the proton restore left the gateway broken: the v4 default route was gone from the netns (mechanism not definitively established; the route is setup-time state that no rotate re-asserted) so the agent failed `NetworkUnreachable`, and a failed warp attempt's v6 address survived because the restore only removed addresses listed in the saved profile (attempt 3's profile was never saved). `rotate_proton` now re-asserts `ip route add default dev freehop0` and scrubs EVERY global address except `WG_ADDR` (plus link-local), not just the saved profile's. `write_state` now replaces the state file instead of merging, so a proton rotate no longer carries stale warp fields (`exit_ip6`, `prefix64`) from a previous warp rotate.
- Fix (found live 24 Nov, user report): `rotate_proton` picked a new server (different EntryIP) and fetched the exit IP once, but never checked the exit IP actually changed, so a rotate could land on (nearly) the same egress IP and the 429 stayed. Two rotates to US-FREE#126 gave 72.251.220.11 then 72.251.220.10, and a single 15s ipify timeout failed rotate attempt 1 after the peer had already moved to US-FREE#127 (attempt 2 then bounced back to US-FREE#126). Proton now verifies the exit IP changed, like tor and warp already did: it records the current exit IP before the swap, retries the exit-IP fetch (3x, 3s apart) instead of failing on one slow ipify, and tries up to 3 lowest-load servers per rotate when the exit IP does not change; a swap failure on one candidate also moves to the next. A rotate only counts when the exit IP is genuinely new. Tested in the sandbox with stubbed servers and a scripted exit-IP sequence (11 checks).
- Status fix (v0.6.1, never tagged separately, ships here): `freehop status` showed the proxy as dead (`http_code 000`, empty `exit_ip`) while `freehop doctor` passed 20/20 right after. Cause: the status proxy check used `curl -m 5`, one attempt, while doctor uses `-m 10`, and a cold TLS handshake through the tunnel (host, veth, netns, Python proxy, WireGuard, Proton) can exceed 5s. Status now uses `-m 10` like doctor and retries once on 000/empty (2s apart); the exit-IP check also retries once on empty. Doctor, watcher, and gateway mode untouched.
- Status fix (v0.6.1, never tagged separately, ships here): the live exit-IP check could still come back empty when the whole proxy path stalled >10s (both retries included), even though the exit was fine. `freehop status` now falls back to the last known exit IP from `rotate-state.json` and says so honestly: new `exit_ip_source` field in `status --json` (`"live"` when the live check answered, `"cached"` when the fallback was used) and `exit IP: x.x.x.x (cached, live check stalled)` in the text output. The `exit_ip` field itself keeps its old meaning (the current exit as best known); the MCP `freehop_status` tool passes the JSON through unchanged, so the added field breaks nothing. If no rotate has ever happened, `exit_ip` stays empty with source `"live"`.

## v0.6.0 (2026-11-23)

- Gateway mode: the gateway can now run on the host's own IP (`GATEWAY_MODE=direct`) with FreeHop as a fallback, instead of always behind the proxy. On a confirmed 429 (probe 429 twice, 120s apart, same debounce) the watcher switches the gateway to the proxy (restores the `freehop-proxy.conf` drop-in, restarts the gateway, no rotate spent because the current exit is healthy) and while on the proxy it tries returning to direct every `RETURN_CHECK_MINS` (default 30) once the gateway is healthy; a failed return restores the proxy. The mode lives in `watch-state.json` (`gateway_mode`) so a restart does not forget it. `freehop status --json` reports `gateway_mode` and `doctor` gains a "gateway mode consistent" check (19 to 20 checks) verifying the recorded mode matches the drop-in on disk. Proven on the live VPS 23 Nov: the host IP (129.226.193.34) answered 200 on a small probe and two large scene generations went through with no 429 at all (both later failed on free-model output validation, a different problem), five days after the same IP was last seen limited. The tunnel stays up in direct mode as the fallback, so doctor still checks it.
- Note: a return attempt costs one gateway restart (two if direct is still limited and the proxy must be restored), at most once per `RETURN_CHECK_MINS`, only while the gateway is healthy on the proxy.

## v0.5.1 (2026-11-23)

- Fix: the Proton session agent died minutes after every connect (`TimeoutError: BrokenPipe` from `Listener.listen()`, ~30-90s after connect, tunnel unaffected) and nothing restarted it, because the agent called `listen()` once and the process exited on the first server-side close. `freehop-agent.py` now reconnects forever with backoff (5s, doubling to 60s, reset after a session that lived >60s) and every log line carries a timestamp (the old log had none, so the death was only visible from the file mtime). The watcher also restarts the agent if the process itself is gone (proton only, current domain from `rotate-state.json`, no peer change, no rotate budget spent), and `freehop doctor` gains an "agent alive" check (18 to 19 checks). Proven live 23 Nov: new agent survived past the old <90s death, reconnecting after each server close, gateway still HTTP 200.
- Note: the agent dying never killed the tunnel (WireGuard keepalive 25s, handshake stayed recent, gateway kept answering 200 for 24+ minutes with the agent dead), so this is a session-hygiene fix, not a tunnel fix.
- Trap found while deploying this fix: stopping the agent is `sudo -n pkill -f freehop-agent.py` (the only form the shipped sudoers file allows), and it matches any caller whose own command text contains that filename, so a long deploy command naming the agent file gets SIGTERMed (exit -15) mid-run. Run `freehop rotate` from a short command instead; production callers (watch timer, CLI) are safe because their command lines never contain the filename.

## v0.5.0 (2026-11-23)

- Watcher now detects the 429 by a direct endpoint probe: `PROBE_CMD` (a command that prints the gateway HTTP status) is the primary 429 signal, because the gateway's journal never logs the 429 -- the 429 only exists in the HTTP response to the caller, so journal grep alone was blind. Proven live 23 Nov: the watcher saw 429 twice, rotated to US-FREE#106, and the gateway was back to HTTP 200. A single 429 never rotates; the probe must answer 429 twice, `PROBE_CONFIRM_SECS=120` seconds apart, so a short 429 flap is ignored. Other statuses (200/502/503/000) never rotate on their own. `examples/test-9router.py` gains `--probe` mode (prints only the status code, always exits 0). `freehop doctor` gains a probe-command check (17 to 18 checks). Post-rotate `TEST_CMD` is retried up to 4x, 12s apart, because the gateway needs 9-37s to listen after a restart. Cost: one tiny completion per 5-minute tick.
- Fix: config values with spaces must be quoted -- `WATCH_SINCE=10 minutes ago` (and any multi-word `POST_ROTATE_HOOK` / `TEST_CMD`) without quotes broke bash sourcing of the config (`config: line 21: minutes: command not found`). The shipped `freehop.conf.example` carried the same bug and is fixed here, with a note in the file.
- Fix: the agent runs under sudo (HOME=/root) inside the namespace, but the keyring and cert live under the invoking user's home, so `freehop-rotate.py` now runs the agent as `sudo -n env HOME=<user home> ip netns exec ...` (sudo strips env set before it).

## v0.4.0 (2026-11-22)

- Egress methods: `EGRESS_METHOD` in the config picks the exit. `proton` (default) keeps the WireGuard flow. `tor` sends `SIGNAL NEWNYM` to the Tor control port and verifies the exit IP changed (up to 3 attempts); the proxy chains through Tor's SOCKS port, managed by `freehop-tor.service`. `proxy-pool` round-robins over `PROXY_POOL` (your own `http://` or `socks5://` upstreams); a dead entry fails the rotate and is skipped next time because the pointer already advanced. The proxy reads `$STATE_DIR/upstream.json` on every connection, so a rotate takes effect without a restart.
- Watcher health check is method-aware: for tor it tests the SOCKS port from inside the namespace instead of a direct egress that can never work there. `freehop status --json` now reports `egress_method`, `upstream`, and `last_rotate.method`; `freehop rotate --json` reports `method`. `doctor` runs the checks for the configured method only. The installer checks the prerequisites for the configured method (Proton keyring only for proton, `tor` only for tor, non-empty pool only for proxy-pool). `EXIT_IP_URL` is now configurable (default `https://api.ipify.org`).
- Fixes found by testing: the proxy used to send a `CONNECT` to an HTTP upstream before a plain http request, which lost the request path (the upstream saw the absolute URI as a raw tunnel and answered the wrong path); plain requests now open a plain connection to the HTTP upstream. And `curl -x` silently bypassed the proxy for hosts in `no_proxy`, which made the exit-IP check and the doctor's proxy check lie; every `curl -x` in the project now passes `--noproxy ''`.
- Not implemented on purpose: WARP (tested on this VPS, both shared egress IPs were already 429) and IPv6 /64 rotation (this VPS has no IPv6 at all, verified). Client-side lockout clearing (the imitated project's `drafts.sqlite` step) is deliberately not copied; FreeHop only rotates the network exit.
- Tested in the sandbox with a local target server, a dummy HTTP upstream, and a fake Tor daemon (SOCKS5 + control port stand-in): http and https through the chain, pool round-robin, dead pool entry failing cleanly, NEWNYM flipping the exit IP, doctor's tor checks, CLI and MCP JSON shapes. Then tested with a real Tor daemon on the VPS (Tor 0.4.9.11, bootstrapped in 15s): the proxy chained through the real Tor SOCKS port, the exit was a real Tor exit (171.25.193.36, IsTor true), `freehop-rotate.py` NEWNYM changed it to 185.220.101.98 on the first attempt (IsTor true), and one combos call through the new exit returned HTTP 200. One deployment caveat found by that test: the tor method as designed (Tor on the host, proxy in the namespace) needs the host firewall to allow the FreeHop subnet to reach the host. On this VPS UFW drops INPUT by default, so a namespace-to-host connection to the Tor SOCKS port times out; the Proton method never needs that direction, which is why it has no such rule. Until that allow exists, the tor method works with the proxy on the host, not in the namespace. Update 22 Nov ~15:10 WIB: the allow was added on this VPS (`ufw allow from 10.200.0.0/24 to any port 9050 proto tcp`, comment "FreeHop Tor SOCKS"), and the tor method was re-verified with the proxy inside a namespace: exit 192.42.116.68 (IsTor true), NEWNYM to 204.8.96.155 (IsTor true) on the first attempt. The in-namespace tor deployment is now proven on this machine.
- The multi-egress idea follows ShadowySphinX/Opencode-Unlimited-Limit-Bypass-, the project openhop is also based on (credited in the README).

## v0.3.4 (2026-11-22)

- README FAQ: 9Router already has proxy pools, FreeHop is the source of fresh exits that fills them, not a competing router.

## v0.3.3 (2026-11-22)

- The 9Router + OpenCode stack is now named everywhere: README intro, skill file description and intro, and the GitHub repo description and topics.

## v0.3.2 (2026-11-22)

- README setup prompt now names the stack it is built for: 9Router serving the OpenCode provider's `combos` model, with the concrete config instead of a generic gateway placeholder.

## v0.3.1 (2026-11-22)

- README: copy-paste setup prompt for AI agents under "AI agents" (skill file first, prerequisites, install, configure for your gateway, verify, MCP if available, safety rules).

## v0.3.0 (2026-11-22)

- Logo rebuilt from zero: the arc-and-dots mark is gone, replaced by a bold geometric F built from hop steps (top bar reaches further than the middle bar). Same emerald tile, same VectorCraft 0.24.1 build, verified at 512px and 32px.

## v0.2.4 (2026-11-22)

- Logo redesigned with the antislop-ui + design-taste-frontend checklists: arc thinner (14, was 18) so the exits read as nodes, middle exit bigger (r22, was 17) as the current exit, mark optically centered (was 20pt low). Still flat emerald, no gradient, no decoration. Rebuilt in VectorCraft 0.24.1, verified at 512px and 32px.

## v0.2.3 (2026-11-22)

- Logo now has a native VectorCraft project: `assets/logo.vectorcraft`, built with VectorCraft 0.24.1's own headless CLI (`vectorcraft-cli run`, 18 command steps: tile, arc, three exits), not hand-written. Rendered and verified with VectorCraft's own renderer; `logo.png` is now VectorCraft's export. README no longer tells anyone to edit anything.

## v0.2.2 (2026-11-22)

- README: logo section tells people to edit `assets/logo.svg` in VectorCraft, with the mark's construction (circles, arc, tile) so it can be rebuilt there.

## v0.2.1 (2026-11-22)

- Logo: `assets/logo.svg` (source) and `assets/logo.png` (512px render). One trajectory hopping through three exits, emerald tile, white mark. README now shows it with badges, and the install and commands sections are current (`--yes`, `~/.local/bin`, `--json`, `freehop mcp`).

## v0.2.0 (2026-11-22)

- MCP server at `src/freehop-mcp.py` (stdio, Python stdlib only, no dependencies). Tools: `freehop_status`, `freehop_doctor`, `freehop_rotate` (`dry_run`), `freehop_logs`. It only invokes the CLI with fixed arguments; teardown is deliberately not exposed.
- `freehop mcp` prints the client config JSON for the machine. Example in `examples/mcp-config.json`.
- Installer now links `freehop` into `~/.local/bin` (uninstaller removes it).

## v0.1.1 (2026-11-22)

- AI-agent friendly: `freehop status`, `doctor`, and `rotate` take `--json` (one JSON object on stdout, progress on stderr), exit 0 ok / 1 failed / 2 usage error, non-interactive install with `--yes`.
- Agent Skills file at `skills/freehop/SKILL.md`: when to use it, JSON schemas, workflows, and safety rules.
- `doctor` checks now carry a fix hint on failure, in both human and JSON output.

## v0.1.0 (2026-11-22)

- Initial public release. Scoped WireGuard netns + forward proxy + watcher that rotates the Proton Free exit on 429 or tunnel death, with rotate limits (10 min / 6 per hour / 20 per day), agent retry for the 86203 race, installer, doctor, and a boot setup unit.
- Tested in an isolated scratch netns on the VPS: setup idempotent, NAT and proxy egress ok, rotate dry-run and a single watch tick against the live tunnel (read-only), clean teardown.
