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
| Direct anti-block | The operator wants direct VPS protocols and no Cloudflare proxy path | DNS-only node domains + VLESS Reality / Trojan Reality / Hysteria2 |

## Workflow

1. Collect deployment inputs before SSH:
   - Main panel VPS: IP, SSH user/port/auth, management domain, subscription domain.
   - Mode: single-point or cluster.
   - Node VPS list for cluster mode: IP, SSH user/port/auth, node domain, management API endpoint.
   - Root domain and DNS provider credentials.
   - Protocols to expose: for Cloudflare path, start with `vless-xhttp-tls`; for direct anti-block mode, use `vless-reality`, `trojan-reality`, and `hysteria2`, and remove XHTTP/Cloudflare, Trojan TLS, and Shadowsocks from the user-facing subscription unless explicitly requested.
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
   - If native 3X-UI subscriptions import but need richer Clash/Mihomo strategy groups and rules, deploy a localhost native profile wrapper in front of `/clash/<subId>`.
   - Deploy Sub-Store only when format conversion, external subscriptions, or multi-source subscription processing is required.
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
- Use a localhost native profile wrapper when the operator wants 3X-UI-managed users, traffic, and nodes, but also needs a full Clash/Mihomo profile with visible `proxy-groups`, `rule-providers`, and `rules`. The wrapper should read the native 3X-UI `/clash/<subId>` source and publish the enhanced profile at the same public `/clash/<subId>` path through Nginx.
- Use Sub-Store only when a client reports `cannot unmarshal !!str` or `cannot unmarshal !!seq`, when external subscription sources are required, or when the operator needs Sub-Store-specific rewrite features.
- Any adapter or native wrapper must forward subscription metadata headers from the native 3X-UI source, especially `subscription-userinfo`, plus `profile-title`, `profile-update-interval`, and `profile-web-page-url` when present. Without `subscription-userinfo`, clients may import nodes but show no traffic quota or expiry.
- For Mihomo clients, include `meta-rules-dat` `rule-providers` and visible strategy groups when the user wants routing rules. For a v2ray-agent-style profile, expose `AUTO`, `自动选择`, `故障转移`, `负载均衡`, `节点选择`, `流媒体`, `手动切换`, `全球代理`, `DNS_Proxy`, `Telegram`, `Google`, `YouTube`, `Netflix`, `Spotify`, `HBO`, `Bing`, `OpenAI`, `ClaudeAI`, `Disney`, `GitHub`, `国内媒体`, `本地直连`, and `漏网之鱼`; route each `RULE-SET` to the matching group. Keep the last rule as `MATCH,漏网之鱼`.
- In cluster mode, run Sub-Store only on the main panel VPS. Do not install it on every remote node.
- In direct anti-block mode, keep node traffic on DNS-only hostnames such as `ca.example.com` and `la.example.com`; do not use Cloudflare orange-cloud hostnames or CF preferred IPs for Reality/Vision/Hysteria2 nodes.
- In direct anti-block mode, the adapted subscription should only expose `vless-reality`, `trojan-reality`, and `hysteria2` direct nodes by default. Disable or filter out `vless-xhttp`, Trojan TLS, and Shadowsocks entries if the goal is a clean direct-only client profile.
- If the client uses fake-ip DNS and proxy server domains resolve to `198.18.0.0/15`, rewrite direct node `server` values to the VPS IPs while preserving Reality `sni`/`servername` and Hysteria2 `sni`. Otherwise the client may dial the fake IP and show `Timeout`.
- If Reality nodes still show intermittent `Timeout` after DNS/IP rewrite and the TCP ports are reachable, test the Reality `target`/`serverNames` before changing unrelated parts. Prefer a stable non-Apple, non-Cloudflare TLS 1.3 target such as `www.yahoo.com:443`/`www.yahoo.com`; avoid assuming `www.cloudflare.com`, `www.microsoft.com`, or Apple/iCloud targets will be stable from every VPS/client path. Re-fetch the Clash profile and run multiple Mihomo delay rounds after changing the target.
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

Adapted Clash/Mihomo, when Sub-Store is deployed:
https://store.example.com/<random-api-prefix>/download/linkray-full?target=ClashMeta&includeUnsupportedProxy=true&prettyYaml=true

Dynamic adapted Clash/Mihomo, when 3X-UI should display the adapted link:
https://sub.example.com/store/<subId>

Enhanced native Clash/Mihomo, when the native wrapper is deployed:
https://sub.example.com/clash/<subId>
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
| Feeding a base64 or JSON-array subscription to a Clash profile importer | Use the native `/clash/<subId>` path; add Sub-Store only if native Clash output still fails |
| Returning only `proxies:` and the client shows no nodes | Wrap the output as a full profile with `proxy-groups` and `rules` |
| Returning rules but no visible strategy groups | Add the v2ray-agent-style groups such as `节点选择`, `流媒体`, `OpenAI`, `ClaudeAI`, `GitHub`, `本地直连`, and `漏网之鱼`, then map `RULE-SET`s to those groups |
| Adapted profile shows no traffic quota or expiry in the client | Forward the native `subscription-userinfo` response header through the wrapper |
| Creating one Sub-Store item per 3X-UI user manually | Use a dynamic `/store/<subId>` route with `fakeSub=1&url=<native-clash-url>` |
| Mixing direct anti-block and CF preferred-IP nodes in one clean mode | Use separate modes; for direct anti-block, expose only Reality/Hysteria2 direct nodes |
| Reality works sometimes but still times out during client delay tests | Verify the advertised server is the VPS IP, then change Reality `target/serverNames` to a stable non-Apple TLS site such as `www.yahoo.com`; if only one VLESS port is flaky, move that inbound to another allowed direct TCP port such as `8443` |
| Saying "all protocols use Reality" | Add Reality profiles where supported, but keep separate usable profiles such as Trojan TLS when full protocol coverage is requested |
| Creating Hysteria2 as TCP+TLS | Set `streamSettings.network=hysteria` and verify UDP listening |
| Adding too many protocols first | Start with VLESS/XHTTP/TLS; add Reality/Hysteria2/Trojan/SS after the base subscription works |
| Renaming 3X-UI internals to a brand | Only rewrite page text at Nginx/Caddy; do not rename services, DBs, API paths, or tags |

## References

- `references/cluster-blueprint.md`: command-level deployment model, DNS/TLS layout, 3X-UI settings, node registration, inbound and user creation rules.
