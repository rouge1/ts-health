# ts-health

A one-screen health summary for a [Tailscale](https://tailscale.com) node.

`tailscale status` tells you who is on your tailnet. It doesn't directly tell
you whether *your* node is healthy, or whether your traffic is actually leaving
through the exit node you selected. `ts-health` answers those questions in one
screen, and verifies the routing claim against the kernel instead of guessing.

```
Tailscale — laptop (100.64.0.1)

  tailscaled     enabled, active
  BackendState   Running
  Health         OK — no warnings reported
  Peers          1/3 online
  Exit node      gateway (100.64.0.2) — online, in use
  Peer path      direct 192.0.2.10:41641, 2ms
  Egress route   via tailscale0 (kernel: ip route get 1.1.1.1)
  Public IP      203.0.113.7
                 note: identical to your own WAN IP when the exit node
                 shares your uplink — the route line above is the proof

  healthy
```

*(Addresses above are RFC 5737 / RFC 6598 documentation ranges, not real hosts.)*

## Why the "Egress route" line exists

The obvious way to check whether an exit node is working is to curl a
what-is-my-IP service and see if the address changed. **That test is unreliable**:
if your exit node sits on the same uplink as you — a second machine on your
LAN, say — your public IP is identical whether the exit node is in use or not.
The check passes while proving nothing.

`ts-health` instead asks the kernel directly:

```
$ ip route get 1.1.1.1
1.1.1.1 dev tailscale0 table 52 src 100.64.0.1
```

Tailscale installs a policy-routing rule (priority 5270) pointing at table 52,
which holds `default dev tailscale0` when an exit node is active. If the kernel
resolves a public destination out `tailscale0`, your traffic is genuinely going
through the tunnel. That is the load-bearing signal in the output. The public IP
line is retained as a convenience and labeled as non-probative.

No packets are sent for this check — `ip route get` is a local query.

## Install

Requires `bash`, `python3`, `iproute2`, `curl`, and `tailscale`. All but
`tailscale` ship with a stock Ubuntu/Debian desktop; on a minimal install you
may need `curl` (or just use `--quick`, which never calls it).

```bash
git clone https://github.com/<you>/ts-health.git
install -m755 ts-health/ts-health ~/.local/bin/ts-health
```

Or symlink it, so `git pull` updates the installed command:

```bash
ln -s "$PWD/ts-health/ts-health" ~/.local/bin/ts-health
```

Make sure `~/.local/bin` is on your `PATH`.

## Usage

```
ts-health            # full check
ts-health --quick    # skip network probes (no peer ping, no IP lookup)
ts-health --help
```

**No `sudo` required.** `tailscale status` is read-only; only mutating commands
(`up`, `set`, `down`) need root. If you are in the habit of typing
`sudo tailscale status`, you can stop.

### Exit codes

| Code | Meaning |
|------|---------|
| `0`  | healthy |
| `1`  | degraded — see the flagged lines |
| `2`  | tailscale not installed, or status unreadable |

Which makes it usable unattended:

```bash
ts-health --quick || notify-send "tailnet degraded"
```

A node is reported **degraded** when any of these hold:

- `tailscaled` is not active
- `BackendState` is not `Running`
- Tailscale reports one or more health warnings
- an exit node is selected but offline
- no pong from the exit node within 12s
- an exit node is set but egress is *not* routing through the tailnet
- the public IP lookup failed

The degraded verdict is a heuristic over those conditions — a flagged line means
*look*, not necessarily that something is broken. `Egress route` is the one to
treat as a hard signal.

## Privacy

The script contains no hardcoded hostnames, addresses, or credentials;
everything identifying is read at runtime. Two constants appear:

- `1.1.1.1` — a route-lookup target only. Never contacted.
- `api.ipify.org` — a third-party IP-echo service, contacted once per full run.
  Use `--quick` to skip it, or swap it for an endpoint you trust.

**Its output, however, contains your public IP, tailnet addresses, LAN
addresses, and device hostnames.** Sanitize before pasting into a bug report or
issue. Note also that raw `tailscale status` output includes the tailnet login
email on every line — `ts-health` deliberately does not print it.

## Compatibility

Linux only. It depends on `systemd` for daemon state and on Tailscale's Linux
policy-routing layout (table 52) for the egress check — neither applies on
macOS or Windows.

## License

MIT — see [LICENSE](LICENSE).
