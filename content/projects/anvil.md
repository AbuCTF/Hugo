---
title: "Anvil"
tagline: "Self-hosted CTF platform"
category: "Security"
status: "active"
type: "platform"
description: "Boot2Root and Attack-Defense CTF platform. Challenges run as Docker containers or full VMs, players reach them over WireGuard."
language: "Go"
stars: 5
tech:
  - "Go"
  - "WireGuard"
  - "Docker"
  - "libvirt"
  - "PostgreSQL"
  - "SvelteKit"
github: "https://github.com/AbuCTF/Anvil"
demo: "https://play.abu.rocks"
logo: "/images/projects/anvil-logo.png"
draft: false
---

Self-hosted Boot2Root and Attack-Defense CTF platform. Currently v0.1.0, still in dev.

## Why

CTFd handles jeopardy CTFs well. B2R and AD competitions need something else: real machines, network reachability, per-player isolation. Anvil is that.

## How players connect

No port bindings, no NAT, no `nc host 31337`. Every player gets a WireGuard keypair and a /32, and reaches challenge IPs on their native ports. Access control happens at the VPN layer.

The client config routes two networks:

```ini
AllowedIPs = 10.100.0.0/16, 172.20.0.0/16
```

`10.100.0.0/16` is the libvirt VM network, `172.20.0.0/16` is the Docker challenge bridge.

Peers are added and removed at runtime. Since the API runs in a container, it reaches the host's WireGuard interface through PID 1's network namespace:

```bash
nsenter -t 1 -n wg set wg0 peer <pubkey> allowed-ips <ip>/32
```

A systemd timer parses `wg show wg0 dump` every five seconds and writes handshake and transfer counters back to the `vpn_configs` table, so the platform knows who is actually online.

## Challenges

Two kinds. Docker containers, capped at two instances per user with a two-hour timeout. And full VMs from OVA, VMDK or QCOW2 images.

VM images get validated before they are trusted: QCOW2 is checked by magic bytes and version, sparse VMDK by its `KDMV` header, and an OVA has to actually contain an `.ovf` descriptor. Uploading a multi-gigabyte image goes through a chunked API (init, PUT chunks, ask which chunks are missing, complete) because a single POST through Cloudflare is not going to survive.

Big VMs run on remote nodes over SSH and virsh. The base image ships once, then each instance is a copy-on-write overlay:

```bash
qemu-img create -f qcow2 -F qcow2 -b base.qcow2 overlay.qcow2
```

## Flags

Every instance gets its own flag. Anvil generates `PREFIX{uuid}` and injects it into the container as `FLAG` and `FLAG_<NAME>`.

That makes flag sharing detectable for free. If you submit a flag that was minted for someone else's instance, the submission is accepted normally and quietly logged to `flag_share_events`. You don't learn that you were caught, and an admin reviews it later.

## Stack

Go 1.24 with Gin for the API, SvelteKit for the web, PostgreSQL 16 and Redis 7. Nine migrations, thirty-odd tables. JWTs are 15-minute access with 7-day refresh, HMAC enforced, passwords bcrypt.

Rate limits are per-endpoint rather than global: 5 logins per 15 minutes, 10 flag submissions per minute, 3 instance starts per 10 minutes, 2 VPN config regenerations per hour.

## Deployment notes

Two things that cost real time, both now in the README.

Docker installs a drop rule in the `raw` PREROUTING chain for bridge IPs, so VPN clients can't reach challenge containers until you accept `172.20.0.0/16` there. And Docker's NAT will masquerade challenge replies back at VPN clients unless you add a `RETURN` in nat POSTROUTING.

Uploads get their own DNS-only `upload.<domain>` vhost with `client_max_body_size 0` and `proxy_request_buffering off`, because Cloudflare caps request bodies at 100MB and VM images are not 100MB.
