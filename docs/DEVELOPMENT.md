# Development

Work inside a disposable Ubuntu VM and snapshot it before loading a module for
the first time: a kernel bug can crash or hang the machine.

## Setup

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r) pkg-config \
     libnl-3-dev libnl-genl-3-dev libcurl4-openssl-dev libcjson-dev
```

## Edit / reload cycle

A loaded module keeps running its old code, so after each change:

```bash
sudo rmmod firewall_module
make -C kernel clean && make -C kernel
sudo insmod kernel/firewall_module.ko
sudo dmesg | tail
```

`modinfo kernel/firewall_module.ko` shows metadata, including `vermagic`,
which must match the running kernel.

## Manual test procedure

1. Load the module; confirm `firewall_module: loaded` in `dmesg`.
2. With no agent running, confirm traffic is unaffected (empty ruleset).
3. Serve a rules response containing an active blacklist entry for a test IP
   (any HTTP server returning the JSON shown in `ARCHITECTURE.md` works) and
   start the agent with `FIREWALL_API_URL` pointing to it.
4. `ping` the test IP: expect 100% loss and `dropping packet` lines in
   `dmesg`.
5. Set the rule to `"active": false` (or remove it): after the next poll the
   ping succeeds again.
6. Stop the API: the agent logs fetch failures but the kernel keeps the last
   good ruleset.
7. `rmmod`/`insmod` the module while the agent runs: the agent re-resolves the
   family and pushes rules again on the next cycle.
8. `sudo rmmod firewall_module`: traffic is fully restored.
