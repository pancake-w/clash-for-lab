# Codex Remote through Mihomo TUN

Use this runbook when Codex works locally but a Remote task repeatedly reports
`Reconnecting... waiting for network`, or when an agent needs to inspect or
change the host-wide proxy path used by Codex.

## Why TUN is required

`clash proxy on` exports `http_proxy`, `https_proxy`, and SOCKS variables for a
shell. A Codex desktop/Remote host process that was launched elsewhere, or was
already running, does not reliably inherit those variables. Pointing a shell at
`127.0.0.1:7890` therefore proves only that applications launched from that
shell can use the proxy.

Mihomo TUN is the host-wide path for this use case. Its automatic routes send
new Codex connections through the `Meta` interface without depending on the
Codex process environment. Existing TCP connections can retain their old route;
restart Codex after TUN is verified when a clean network cutover is required.

## Security boundary

TUN creation and route changes require `CAP_NET_ADMIN`. Grant it only to the
trusted Mihomo binary installed by this project:

```bash
sudo setcap cap_net_admin=+ep "$HOME/tools/mihomo/bin/mihomo"
getcap "$HOME/tools/mihomo/bin/mihomo"
```

The expected result contains `cap_net_admin=ep`. This capability lets that
binary create network interfaces and modify routes. Re-check it whenever the
Mihomo binary is replaced or upgraded; file capabilities are attached to the
binary file and may be lost on replacement.

Never commit a subscription URL, controller secret, proxy-node credential, or
the full generated `runtime.yaml`.

## Supported activation path

Use the repository commands as the source of truth:

```bash
clash status
clash tun status
clash tun on
```

`clash tun on` persists the TUN preference and restarts Mihomo as needed. If
TUN startup fails, the command must return failure and restore the disabled
preference; this rollback is covered by `tests/lifecycle-regressions.sh`.

Run `clash on` only when `clash status` says Mihomo is stopped. It is unnecessary
when Mihomo is already running and the TUN checks below pass. Shell proxy
variables may still be enabled for terminal programs with `clash proxy on`, but
they are not the mechanism that guarantees routing for Codex Remote.

## Verification order

Run these checks from an independent terminal on the connected host:

```bash
clash status
clash tun status
getcap "$HOME/tools/mihomo/bin/mihomo"
ip -brief link show Meta
ip route get 1.1.1.1
```

The state is ready for Codex when all of the following are true:

1. Mihomo is running and its connectivity check succeeds.
2. TUN is enabled in the runtime configuration.
3. `getcap` reports `cap_net_admin=ep`.
4. The `Meta` interface is `UP`/`LOWER_UP`.
5. The route probe reports `dev Meta`.

Treat the interface name and `dev Meta` route as the decisive host-wide signal;
proxy environment variables alone are insufficient.

## Restart Codex after the route is ready

A complete restart interrupts every active Codex task on this host. Run it from
a separate local or SSH terminal, never from the Codex task that must survive:

```bash
pkill -KILL -x codex
pkill -KILL -f '/codex-code-mode-host$'
sleep 2

pgrep -a -x codex
pgrep -af '/codex-code-mode-host$'
```

Both checks should initially print nothing. Reopen the Codex host application
and reconnect the existing task. Then confirm that all process IDs and start
times are new:

```bash
ps -C codex -o pid,ppid,lstart,etime,stat,args
pgrep -af '/codex-code-mode-host$'
```

A Remote disconnect during the restart is expected. Do not delete the task or
its workspace; reconnect it after the host returns.

## `robot-s0` reference state

This is a diagnostic snapshot, not a portable default. On 2026-09-25 the
working host had:

- installation directory: `$HOME/tools/mihomo`
- mixed proxy: `127.0.0.1:7890`
- controller: `127.0.0.1:9090`
- DNS listener: `127.0.0.1:15353`
- TUN: enabled with automatic routing and interface detection
- TUN interface: `Meta`
- Mihomo capability: `cap_net_admin=ep`
- process wrapper: transient user unit
  `clash-for-lab-mihomo-tun.service`

The transient unit was used for the live recovery and is not repository-managed
or persistent configuration. Agents must not assume it exists after logout or
reboot. Diagnose using the verification order above and restore service through
`clash on` / `clash tun on` rather than recreating an undocumented unit.

## Recovery

To leave host-wide TUN routing and return to shell-only proxying:

```bash
clash tun off
clash proxy on
```

To remove the network capability as well:

```bash
sudo setcap -r "$HOME/tools/mihomo/bin/mihomo"
```

After any recovery or activation change, repeat the verification order instead
of inferring success from a single status line.
