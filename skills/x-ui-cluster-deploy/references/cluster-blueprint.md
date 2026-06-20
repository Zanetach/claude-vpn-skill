# 3X-UI Cluster Deployment Blueprint

Use this reference after `SKILL.md` triggers. It describes the target state and command-level rules for a main-panel 3X-UI cluster with one subscription URL.

## Target State

Two VPS example:

```text
VPS-A
  3x-ui main panel
  subscription server
  local node1 inbounds

VPS-B
  3x-ui remote panel/node
  node2 inbounds

User imports only:
  https://sub.example.com/sub/<subId>
```

DNS:

```text
panel.example.com  A/AAAA  VPS-A
sub.example.com    A/AAAA  VPS-A
node1.example.com  A/AAAA  VPS-A
node2.example.com  A/AAAA  VPS-B
```

Use Cloudflare orange-cloud only where the selected protocol transport supports it. VLESS XHTTP over TLS can sit behind Cloudflare; direct Trojan or Shadowsocks usually should be DNS-only unless separately wrapped/terminated.

## 1. Install 3X-UI

Install 3X-UI on every VPS, pinning a release if repeatability matters:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

For unattended installs, 3X-UI supports `XUI_NONINTERACTIVE=1` and writes credentials to `/etc/x-ui/install-result.env`. Read that file as root and store secrets outside chat logs.

Use the same 3X-UI version on all nodes when possible.

## 2. Main Panel Settings

On VPS-A, configure the panel as the control plane:

```text
Web domain: panel.example.com
Subscription enabled: true
Subscription listen: 127.0.0.1
Subscription port: 10882
Subscription domain: sub.example.com
Sub path: /sub/
JSON path: /json/
Clash path: /clash/
Clash enable: true when Clash/Mihomo users exist
```

If editing SQLite directly, stop `x-ui`, update `settings`, then restart. Prefer UI/API where available because setting keys may change between releases.

Never apply a blanket `subEnable=false` on the main panel.

## 3. Reverse Proxy

Terminate HTTPS for public names with Nginx/Caddy. Keep 3X-UI web and subscription listeners on localhost where possible.

Nginx pattern for subscription:

```nginx
server {
    listen 443 ssl http2;
    server_name sub.example.com;

    ssl_certificate /root/cert/fullchain.cer;
    ssl_certificate_key /root/cert/example.com.key;

    location /sub/ {
        proxy_pass http://127.0.0.1:10882/sub/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /json/ {
        proxy_pass http://127.0.0.1:10882/json/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /clash/ {
        proxy_pass http://127.0.0.1:10882/clash/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 4. Remote Node API Access

3X-UI nodes are remote 3X-UI panels. The main panel polls each node's API, including `/panel/api/server/status`, with the node's API token.

Preferred access:

```text
VPS-A <-> VPS-B over WireGuard/Tailscale/private VPC
Node address in main panel: remote private IP
Node panel port: private only
```

Acceptable public fallback:

```bash
# On VPS-B, allow only main panel IP to reach panel API port.
ufw allow from <VPS_A_IP> to any port <XUI_PANEL_PORT> proto tcp
ufw deny <XUI_PANEL_PORT>/tcp
```

Do not set the remote node panel `webListen` to `127.0.0.1` unless VPS-A reaches it through SSH tunnel, WireGuard loopback routing, or another explicit private path.

When adding a node in the main panel, fill:

```text
Name: node2
Scheme: https
Address: node2 private IP or management domain
Port: remote 3X-UI panel port
Base Path: remote panel base path
API Token: remote panel API token
TLS verify mode: verify, pin, or mTLS; avoid skip except for first smoke test
Inbound sync mode: all for small clusters; selected for production
```

Click test/probe and require `online` before creating remote inbounds.

## 5. Inbounds Per Node and Protocol

Start minimal:

```text
node1-vless-xhttp-tls
node2-vless-xhttp-tls
```

Add optional protocols only when requested:

```text
node1-trojan-tls
node2-trojan-tls
node1-ss-2022
node2-ss-2022
```

Use distinct tags/remarks. If using Cloudflare, keep protocol/transport compatibility in mind:

| Protocol | Good default | Notes |
|---|---|---|
| VLESS | XHTTP + TLS + Nginx path | Best fit for Cloudflare/CDN fronting |
| Trojan | TLS direct | Usually DNS-only unless fallback/SNI is designed |
| Shadowsocks 2022 | Direct port | Do not route through Cloudflare HTTP proxy |

For remote node inbounds, create or sync them from the main panel so the main database knows their `node_id` and can include them in subscriptions.

## 6. Users and Subscription Aggregation

The aggregation key is `subId`.

Create a client once per user identity:

```text
email/remark: user001
subId: user001_<random>
totalGB: quota bytes or 0 for unlimited
expiryTime: unix ms or 0 for unlimited
limitIp: concurrent IP cap or 0
enable: true
```

Attach that same client identity to every inbound that should appear in the subscription:

```text
user001_<random>
  node1-vless-xhttp-tls
  node2-vless-xhttp-tls
  node1-trojan-tls
  node2-trojan-tls
```

Do not create unrelated subIds per protocol. That fragments the subscription and makes quota/expiry management inconsistent.

## 7. Verification

Before handing over the subscription:

```bash
# On every VPS
systemctl is-active x-ui
ss -tlnp | grep -E ':(443|10882|<panel-port>) '

# From VPS-A to each remote node
curl -fsS -H "Authorization: Bearer <NODE_API_TOKEN>" \
  https://<node-management-host>:<panel-port>/<base-path>/panel/api/server/status

# Public subscription smoke tests
curl -I https://sub.example.com/sub/<subId>
curl -I https://sub.example.com/clash/<subId>
curl -I https://sub.example.com/json/<subId>
```

Then import the Clash/Mihomo subscription in a client and verify it contains every expected node/protocol remark exactly once.

## 8. Output Format

Return:

```text
Main panel: https://panel.example.com/<basePath>
Nodes: node1 online, node2 online
User: user001
Generic subscription: https://sub.example.com/sub/<subId>
Clash/Mihomo: https://sub.example.com/clash/<subId>
JSON: https://sub.example.com/json/<subId>
Included profiles:
  - node1-vless
  - node2-vless
  - node1-trojan
  - node2-trojan
Security:
  - Node API reachable only from VPS-A/private network
  - Subscription public over HTTPS
  - Panel admin access restricted
```
