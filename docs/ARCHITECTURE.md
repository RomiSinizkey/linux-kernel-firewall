# Architecture

## Components

| Component | Path | Role |
|---|---|---|
| Kernel module | `kernel/firewall_module.c` | Netfilter hook, rule table, Generic Netlink handler |
| Agent | `firewall-agent/main.c` | Polls the rules API, validates, pushes rules to the kernel |
| Protocol | `shared/firewall_protocol.h` | Family name, commands, attributes, rule enums (used by both sides) |
| Deployment | `deploy/` | systemd service, module auto-load, installer |

## Packet path

`firewall_hook_fn()` is registered with `nf_register_net_hook()` on
`init_net` for `NFPROTO_IPV4` at `NF_INET_PRE_ROUTING` with priority
`NF_IP_PRI_FIRST`, so it sees every incoming IPv4 packet before routing.
It reads `saddr`/`daddr` from `ip_hdr(skb)`, scans the active ruleset under a
read lock, and returns `NF_DROP` on a match (logging with
`pr_info_ratelimited`) or `NF_ACCEPT` otherwise. A missing skb or IP header
results in `NF_ACCEPT`.

## Rule updates

The module registers a Generic Netlink family named `firewall` (version 1)
with a single admin-only command, `FIREWALL_CMD_REPLACE_RULES`. The message
carries a nested `FIREWALL_A_RULE_LIST`; each entry has a type
(`FIREWALL_RULE_TYPE_IP`), a mode (`FIREWALL_RULE_MODE_BLACKLIST`) and a
`__u32` IPv4 address in network byte order.

The handler validates every entry into a local array first (rejecting
unsupported types/modes, missing fields, or more than `FIREWALL_MAX_RULES`
entries) and only then swaps the new ruleset in under the write lock. An
empty list clears the ruleset.

## Agent cycle

Each cycle the agent: resolves the netlink family id (it changes across
module reloads) -> `GET`s the rules URL -> parses `ips.blacklist`, keeping
active rules only -> sends `REPLACE_RULES` and waits for the kernel ACK. Any
failure before the send leaves the kernel's ruleset unchanged; the agent logs
and retries on the next cycle. `SIGINT`/`SIGTERM` trigger a clean shutdown.

Expected API response:

```json
{ "ips": { "blacklist": [ { "id": 1, "type": "ip", "mode": "blacklist",
                            "value": "203.0.113.7", "active": true } ],
           "whitelist": [] } }
```

## Deployment

`deploy/install.sh` installs the `.ko` into `/lib/modules/$(uname -r)/extra/`,
runs `depmod`, adds `/etc/modules-load.d/firewall.conf`, installs the agent to
`/usr/local/bin`, and enables `firewall-agent.service`. The service reads
`/etc/default/firewall-agent` (see `deploy/firewall-agent.env.example`) and
restarts on failure.
