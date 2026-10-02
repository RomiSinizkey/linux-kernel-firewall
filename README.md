# Linux Kernel Firewall

## Overview

A small IPv4 packet firewall built as a Linux kernel module, plus a user-space
agent that keeps its rules in sync with a REST API.

The kernel module registers a **Netfilter** hook at `NF_INET_PRE_ROUTING` and
drops any incoming IPv4 packet whose source or destination address is on the
active blacklist. The blacklist is not hardcoded: a user-space daemon
(`firewall-agent`) polls a rules API and pushes the full ruleset into the
kernel over **Generic Netlink**.

## Architecture

```text
  Rules API (Node.js, HTTP/JSON)
            |
            |  GET /api/firewall/rules?type=ip   (polled every N seconds)
            v
  +---------------------+
  |   firewall-agent    |   user space (C, libcurl, cJSON, libnl)
  +---------------------+
            |
            |  Generic Netlink family "firewall"
            |  FIREWALL_CMD_REPLACE_RULES (whole ruleset, atomic swap)
            v
  +---------------------+
  |   firewall_module   |   kernel space
  |   rule table        |
  +---------------------+
            ^
            |  lookup
  incoming IPv4 packet --> Netfilter NF_INET_PRE_ROUTING --> firewall_hook_fn()
                                                 match    -> NF_DROP
                                                 no match -> NF_ACCEPT
```

More detail: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Key Features

- Loadable kernel module with a Netfilter `NF_INET_PRE_ROUTING` hook
  (`NF_IP_PRI_FIRST`)
- Drops IPv4 packets whose source **or** destination address is blacklisted
- Up to 128 IPv4 rules, protected by a reader/writer lock
- Runtime rule updates over Generic Netlink (admin-only, `GENL_ADMIN_PERM`);
  each update replaces the whole ruleset atomically, and an invalid message
  leaves the current ruleset untouched
- Strict validation of the netlink message in the kernel and of the JSON in
  the agent
- Rate-limited kernel logging of dropped packets
- Agent re-resolves the netlink family every cycle, so it recovers when the
  module is loaded late or reloaded
- systemd unit, `modules-load.d` entry and install script for start at boot

## Technologies

C, Linux kernel module (Kbuild), Netfilter, Generic Netlink, libnl-3,
libcurl, cJSON, systemd, Make.

## Build and Run

Requirements (Ubuntu): `build-essential`, `linux-headers-$(uname -r)`,
`pkg-config`, `libnl-3-dev`, `libnl-genl-3-dev`, `libcurl4-openssl-dev`,
`libcjson-dev`.

> Kernel modules run with full privileges. Use a disposable VM, not your
> host machine.

```bash
# Build
make -C kernel
make -C firewall-agent

# Load the module and check it
sudo insmod kernel/firewall_module.ko
lsmod | grep firewall_module
sudo dmesg | grep firewall_module

# Run the agent (rules API URL is configurable)
FIREWALL_API_URL="http://<host>:3000/api/firewall/rules?type=ip" \
  ./firewall-agent/firewall-agent

# Unload
sudo rmmod firewall_module
```

Until the agent delivers a ruleset the module accepts all traffic.

Environment variables for the agent:

| Variable | Default |
|---|---|
| `FIREWALL_API_URL` | `http://localhost:3000/api/firewall/rules?type=ip` |
| `FIREWALL_POLL_INTERVAL_SECONDS` | `10` |

### Start at boot (systemd)

```bash
sudo bash deploy/install.sh
sudo cp deploy/firewall-agent.env.example /etc/default/firewall-agent
sudoedit /etc/default/firewall-agent     # set FIREWALL_API_URL
sudo systemctl start firewall-agent
```

## Testing / Verification

- `dmesg -w` shows `replaced active ruleset (N IP rule(s) active)` when the
  agent pushes rules, and `dropping packet (src ..., dst ...)` for each
  (rate-limited) drop.
- `ping <blacklisted-ip>` should show 100% loss while the rule is active and
  succeed again after the rule is deactivated or the module is unloaded.
- `sudo tcpdump -ni any host <ip>` confirms whether packets reach the host.
- `journalctl -u firewall-agent -f` shows the agent's sync cycles.

There is no automated test suite; see [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)
for the manual test procedure.

## Project Structure

```text
.
├── kernel/            # firewall_module.c + Kbuild Makefile
├── firewall-agent/    # user-space sync daemon (main.c + Makefile)
├── shared/            # firewall_protocol.h: netlink contract shared by both
├── deploy/            # install.sh, systemd unit, modules-load.d, env template
└── docs/              # architecture and development notes
```

## Limitations

- IPv4 only; matching is by address only (no ports, protocols or connection
  tracking). One blacklist is matched against both source and destination.
- Maximum of 128 rules.
- Only blacklist mode is implemented; the `PORT` rule type and whitelist
  data in the API response are reserved/ignored.
- The agent uses plain HTTP polling with no authentication or TLS
  configuration; run it only against a trusted API.
- The hook is registered in the initial network namespace only.
- Tested on Ubuntu VMs; no automated tests.
