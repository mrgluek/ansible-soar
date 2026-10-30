# ansible-soar

Ansible playbook for a [soar](https://github.com/getlantern/soar) DNS tunnel
server: clients on restrictive networks, where DNS is often the only thing
that still gets out, reach chatmail relays (Delta Chat / Velta Tunnel)
through DNS queries for a delegated zone.

One variable per server: the zone.

```sh
ansible-playbook -i inventory.ini soar_setup.yml -l ns.example.net
```

## What it sets up

| Piece | Why |
|---|---|
| `soar-server` v0.2.0 (`/usr/local/bin`), built on the control machine | the tunnel; built locally because servers are small and may lack Go |
| `soar.service`, user `soar`, UDP 5302 | runs unprivileged; public port 53 is redirected to it |
| `/etc/soar/key` (root:soar 0640) | generated on the server on first run and never replaced |
| `soar-firewall` + `soar-firewall.service` | 53 → 5302 redirect; chain `SOAR_OUT` lets soar reach only the relays (ipset `soar_relays`) on mail ports and the resolvers from `resolv.conf`, plus the networks from `extra.txt`; over IPv6 only those networks; `soar-firewall-resolv.path` re-applies it when `resolv.conf` changes |
| `/etc/soar/relays.txt` + `soar-relays-update` + timer (30 min) | relay hostnames → ipset, entries expire after a day without refresh |
| `stats.nft` + `soar-stats` + timer (10 min) | per-resolver traffic counters in SQLite (`soar-stats -d 7`); optional |
| `soar-extra.txt` next to the inventory (optional) → `/etc/soar/extra.txt` | fixed networks + ports soar may also reach (ipsets `soar_extra`, `soar_extra6`) |
| `soar-asn-labels.txt` next to the inventory (optional) | extra names for resolver networks in `soar-stats` reports |

The firewall does not use iptables-persistent: apt removes it whenever
something installs ufw, and the next reboot comes up without rules. Rules are
added next to whatever else is there (ufw, docker, fail2ban) and tagged with
the comment `soar`, so `soar-firewall stop` and re-runs touch nothing else.

## New server

1. Server: Debian/Ubuntu with a working resolver in `/etc/resolv.conf`
   (systemd-resolved, Unbound, …) and nothing public on UDP 53. A chatmail
   relay on the same host is fine.
2. Inventory: `[soar]` group, `soar_zone=<zone>`; relay list next to it (see below).
3. Run the playbook. The summary prints the public key and the Velta link
   `velta://tunnel?v=1&zone=…&key=…`.
4. DNS in the parent zone (Cloudflare: DNS only, not proxied):
   ```
   <zone>      NS  <server name>
   <server>    A   <server IP>
   ```
   The playbook checks the delegation and says what is missing.

## Moving a tunnel to another server

Phones keep their link if zone and key stay the same: put the old
`/etc/soar/key` into a vault as `soar_key`, run the playbook on the new host
with the same `soar_zone`, then point the zone's NS at the new server.

## Health check

```sh
systemctl status soar soar-firewall
iptables -t nat -S PREROUTING | grep soar      # 53 -> 5302 redirect
dig @77.88.8.8 x.<zone> TXT                    # NOERROR, flags aa
soar-stats -d 1
```

## Relay list

`soar-relays.txt` next to your inventory (or `soar_relays_file=<path>`), one
hostname per line; start from `files/relays.example.txt`. It is deliberately
not part of this repo: a published list of the relays tunnel users depend on
is a ready-made block list. Edit it and re-run the playbook; on a server,
`soar-relays-update` re-resolves it immediately.

## Extra networks

`soar-extra.txt` next to your inventory (or `soar_extra_file=<path>`): lines of
`<network> <port>[,<port>...]`, IPv4 or IPv6, TCP. Use it for services that are
not chatmail relays but should work through the tunnel, e.g. Telegram's data
centres on 443 (see `files/extra.example.txt`). Everything else stays refused.
Re-running the playbook re-applies the rules without restarting soar; without
the file, `/etc/soar/extra.txt` is removed and so is the access.

## License

MIT
