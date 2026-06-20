---
name: LinkRay-deploy
description: Use when deploying or operating a 3X-UI or LinkRay-branded single-point or cluster setup with a main panel, local or remote VPS nodes, Reality, Hysteria2, XHTTP, user/client creation, subscription aggregation, Clash/Mihomo subscriptions, node failover, or one subscription URL across multiple Xray inbounds.
---

# LinkRay Deploy

## Overview

Deploy and operate a LinkRay-branded 3X-UI single-point or cluster setup where one main panel publishes a single subscription URL and local or remote VPS nodes host the actual proxy inbounds. Use this for "one VPS with subscription", "two VPS nodes", "multiple protocols", "Reality", "Hysteria2", "create users", "single subscription", "multi-node 3x-ui", "LinkRay panel name", and similar requests.

This is a subscription-centered workflow, not the quick single-node `x-ui-deploy` workflow. Do not disable subscriptions. Remote node panels must not be localhost-only unless a private overlay network or tunnel makes them reachable from the main panel.

## Deployment Modes

| Mode | Use when | Shape |
|---|---|---|
| Single-point | One VPS should host panel, subscription, users, and all inbounds | VPS-A: main panel + local node + `sub.example.com` |
| Cluster | Two or more VPS nodes should appear under one subscription | VPS-A: main panel + optional local node; VPS-B/N: remote nodes |

## Workflow

1. Collect deployment inputs before SSH:
   - Main panel VPS: IP, SSH user/port/auth, management domain, subscription domain.
   - Mode: single-point or cluster.
   - Node VPS list for cluster mode: IP, SSH user/port/auth, node domain, management API endpoint.
   - Root domain and DNS provider credentials.
   - Protocols to expose: start with `vless-xhttp-tls`; add `vless-reality`, `trojan-reality`, `hysteria2`, `trojan-tls`, and `shadowsocks-2022` only when requested and client support is clear.
   - Users: email/remark, quota, expiry, IP limit, and `subId` policy.
   - Security choice for node API reachability in cluster mode: WireGuard/Tailscale/private network preferred; otherwise firewall allow only the main panel IP.

2. Read `references/cluster-blueprint.md` before executing commands or changing a server.

3. Deploy in this order:
   - Install 3X-UI on all VPS nodes.
   - Configure DNS and TLS for `panel`, `sub`, and each `nodeN` domain.
   - Enable BBR on every VPS before traffic testing.
   - Configure the main panel subscription server and reverse proxy.
   - In cluster mode, configure remote node API access so the main panel can call each node.
   - In cluster mode, register remote nodes in the main panel and verify heartbeat.
   - Create inbounds per node and protocol.
   - Create clients once on the main panel and attach the same client identity/subId to every intended inbound.
   - Configure UFW and Fail2Ban after SSH access and API reachability are verified.
   - If branding is requested, change only the HTTPS reverse-proxy presentation layer, not the `x-ui` service name, database path, API paths, or node sync identifiers.
   - Verify certificate auto-renewal, subscription decoding, and remote-node API access.
   - Output the subscription URLs, not a pile of separate links.

## Architecture

```text
Clients
  -> https://sub.example.com/sub/<subId>
      -> 3X-UI main panel subscription server
          -> node1 inbounds: VLESS / Trojan / Shadowsocks
          -> node2 inbounds: VLESS / Trojan / Shadowsocks
```

Recommended two-VPS layout:

| Role | Host | Public names |
|---|---|---|
| Main panel + local node | VPS-A | `panel.example.com`, `sub.example.com`, `node1.example.com` |
| Remote node | VPS-B | `node2.example.com`, private/API management endpoint |

Recommended single-point layout:

| Role | Host | Public names |
|---|---|---|
| Main panel + local node + subscription | VPS-A | `panel.example.com`, `sub.example.com`, `node1.example.com` |

## Hard Requirements

- Keep one subscription authority: the main panel.
- Enable the subscription server on the main panel.
- Do not use the single-node hardening that sets `subEnable=false`.
- In cluster mode, remote node panels must be reachable from the main panel by API token, mTLS, pinned HTTPS, or a private overlay route.
- In cluster mode, remote node panel/API ports must not be open to the world. Restrict by firewall or private networking.
- Use one stable `subId` per user across all selected inbounds and protocols.
- Use unique remarks/tags per node/protocol so subscriptions are readable, such as `node1-vless`, `node2-trojan`.
- Use DNS-01 certificates for Cloudflare-managed domains when possible, especially wildcard `example.com` + `*.example.com`.
- Keep subscription service behind a reverse proxy and set `subDomain`; remember direct localhost tests need `Host: sub.example.com` or they will 403.
- If the panel UI should hand users a Clash/Mihomo-ready link, keep `subPath=/sub/` but set the externally displayed `subURI` to `https://sub.example.com/clash/`. Do not set `subPath=/clash/`; that collides with the native Clash route.
- Treat Reality as a transport security option, not a universal wrapper. Use it with TCP VLESS/Trojan inbounds. If the operator wants full protocol coverage, a single subscription can include both Trojan TLS and Trojan Reality as separate profiles. Do not force Reality onto Hysteria2 or Shadowsocks.
- Hysteria2 must use Xray `protocol=hysteria` with `settings.version=2` and `streamSettings.network=hysteria`; TCP+TLS on the same port is not Hysteria2.
- Persist BBR as `net.core.default_qdisc=fq` and `net.ipv4.tcp_congestion_control=bbr` where the kernel supports it.
- Configure Fail2Ban's `sshd` jail for the actual SSH port, not only the default port 22.
- Prefer PostgreSQL on the main panel if managing many clients or many nodes; SQLite is acceptable for a small two-node personal deployment.

## Subscription Output

For each user, output:

```text
Generic:
https://sub.example.com/sub/<subId>

Clash/Mihomo:
https://sub.example.com/clash/<subId>

JSON:
https://sub.example.com/json/<subId>
```

Do not present panel-exported internal links as the primary deliverable. The cluster deliverable is the subscription URL plus admin notes.

## Common Mistakes

| Mistake | Correct action |
|---|---|
| Running the original single-node skill unchanged | Use this skill; original disables subscriptions and localhost-binds the panel |
| Treating single-point as no-subscription | Single-point still enables the main subscription server |
| Making both VPS share one DNS name | Use distinct node domains or host overrides; avoid random DNS routing |
| Creating separate users per protocol | Create one user/subId and attach it to every selected inbound |
| Exposing node panel ports publicly | Use private networking or firewall allow only the main panel IP |
| Making users manually edit `/sub/` to `/clash/` | Configure the displayed `subURI` to `/clash/` while leaving `subPath=/sub/` |
| Saying "all protocols use Reality" | Add Reality profiles where supported, but keep separate usable profiles such as Trojan TLS when full protocol coverage is requested |
| Creating Hysteria2 as TCP+TLS | Set `streamSettings.network=hysteria` and verify UDP listening |
| Adding too many protocols first | Start with VLESS/XHTTP/TLS; add Reality/Hysteria2/Trojan/SS after the base subscription works |
| Renaming 3X-UI internals to a brand | Only rewrite page text at Nginx/Caddy; do not rename services, DBs, API paths, or tags |

## References

- `references/cluster-blueprint.md`: command-level deployment model, DNS/TLS layout, 3X-UI settings, node registration, inbound and user creation rules.
