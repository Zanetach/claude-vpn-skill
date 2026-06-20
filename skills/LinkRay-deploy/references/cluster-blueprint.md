# LinkRay Deployment Blueprint

Use this reference after `SKILL.md` triggers. It describes the target state and command-level rules for a LinkRay-branded 3X-UI single-point or cluster deployment with one subscription URL.

## Target State

Single-point example:

```text
VPS-A
  3x-ui main panel
  subscription server
  local node1 inbounds

User imports only:
  https://sub.example.com/sub/<subId>
```

DNS:

```text
panel.example.com  A/AAAA  VPS-A
sub.example.com    A/AAAA  VPS-A
node1.example.com  A/AAAA  VPS-A
```

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

Use Cloudflare orange-cloud only where the selected protocol transport supports it. VLESS XHTTP over TLS can sit behind Cloudflare; Reality, Hysteria2, direct Trojan, and Shadowsocks should use DNS-only direct hostnames unless a separate compatible fronting layer is deliberately designed.

Choose single-point when the user has one VPS or explicitly asks for "单点". Choose cluster when the user has two or more VPS nodes or asks for remote-node failover/aggregation.

## 0. Cloudflare DNS Layout

For a two-VPS Cloudflare-backed deployment, create records before issuing certificates:

```text
panel.example.com  A  VPS-A-IP  DNS only
sub.example.com    A  VPS-A-IP  DNS only
node1.example.com  A  VPS-A-IP  Proxied when using VLESS/XHTTP over TLS
node2.example.com  A  VPS-B-IP  Proxied when using VLESS/XHTTP over TLS
direct1.example.com A VPS-A-IP  DNS only for Reality/Trojan/Hysteria2/SS
direct2.example.com A VPS-B-IP  DNS only for Reality/Trojan/Hysteria2/SS
```

Use DNS-only for `panel` and `sub` during first deployment. `sub` can remain DNS-only; only proxy it if the subscription domain is intentionally fronted by Cloudflare and tested with the selected clients.

Do not point Reality or Hysteria2 links at orange-cloud hostnames. In subscriptions, use the `nodeN.example.com` orange-cloud hosts only for XHTTP profiles and use direct DNS-only hostnames for TCP/UDP direct protocols.

Use Cloudflare API tokens with least privilege:

```text
Zone:DNS:Edit
Zone:Zone:Read
Scope: selected zone only
```

Verify the token before running acme.sh or DNS automation:

```bash
curl -fsS -H "Authorization: Bearer $CF_Token" \
  https://api.cloudflare.com/client/v4/user/tokens/verify
```

## 1. Install 3X-UI

Install 3X-UI on every VPS, pinning a release if repeatability matters:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

For unattended installs, 3X-UI supports `XUI_NONINTERACTIVE=1` and writes credentials to `/etc/x-ui/install-result.env`. Read that file as root and store secrets outside chat logs.

Use the same 3X-UI version on all nodes when possible.

Recommended unattended install shape for small clusters:

```bash
XUI_NONINTERACTIVE=1 \
XUI_PANEL_PORT=2053 \
XUI_SSL_MODE=none \
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

For the main panel, bind the panel to localhost after install:

```bash
x-ui setting -listenIP 127.0.0.1
systemctl restart x-ui
```

For a public fallback remote-node API, the remote node panel may listen on `0.0.0.0`, but firewall it so only VPS-A can reach the panel port.

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

When Clash/Mihomo is the primary target client, avoid making users manually change `/sub/` to `/clash/`. Keep the route paths distinct, but make the panel's displayed generic subscription URI point at the Clash route:

```text
subPath=/sub/
subURI=https://sub.example.com/clash/
subJsonPath=/json/
subJsonURI=https://sub.example.com/json/
subClashPath=/clash/
subClashURI=https://sub.example.com/clash/
```

Do not set `subPath=/clash/`; 3X-UI already registers `/clash/<subId>`, and duplicate routes can crash the panel at startup.

If editing SQLite directly, stop `x-ui`, update `settings`, then restart. Prefer UI/API where available because setting keys may change between releases.

Never apply a blanket `subEnable=false` on the main panel.

3X-UI v3 API examples use the panel base path:

```bash
BASE="/<main-base-path>"
API="http://127.0.0.1:2053${BASE}/panel/api"

curl -fsS -X POST \
  -H "Authorization: Bearer <MAIN_API_TOKEN>" \
  "$API/setting/all"
```

`/panel/api/setting/all` and `/panel/api/setting/update` are POST endpoints. Do not diagnose them as broken just because GET returns 404.

When `subDomain` is set, the subscription server rejects wrong Host headers with 403. Local tests must include the Host header:

```bash
curl -H 'Host: sub.example.com' http://127.0.0.1:10882/sub/<subId>
```

Without the Host header, a 403 is expected and does not mean the `subId` is missing.

## 2.1 TLS Certificates and Auto-Renewal

For Cloudflare-managed domains, prefer DNS-01 wildcard certificates. Install on every VPS that terminates HTTPS:

```bash
export CF_Token='<cloudflare-api-token>'
export CF_Email='<account-email>'

curl -fsSL https://get.acme.sh | sh -s email="$CF_Email"
/root/.acme.sh/acme.sh --set-default-ca --server letsencrypt
/root/.acme.sh/acme.sh --issue --dns dns_cf \
  -d example.com -d '*.example.com' --keylength ec-256

mkdir -p /etc/ssl/example
/root/.acme.sh/acme.sh --install-cert -d example.com --ecc \
  --fullchain-file /etc/ssl/example/fullchain.cer \
  --key-file /etc/ssl/example/example.com.key \
  --reloadcmd 'systemctl reload nginx || true'
chmod 600 /etc/ssl/example/example.com.key
```

acme.sh installs a cron entry automatically. Verify it:

```bash
/root/.acme.sh/acme.sh --list
crontab -l | grep acme.sh
```

The `Renew` time from `acme.sh --list` is the next renewal window, not the certificate expiry date. Certificates should renew automatically before expiry and reload Nginx through the install hook.

## 3. Reverse Proxy

Terminate HTTPS for public names with Nginx/Caddy. Keep 3X-UI web and subscription listeners on localhost where possible.

Nginx pattern for the main panel, subscription, and local XHTTP node:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;
    listen [::]:80;
    server_name panel.example.com sub.example.com node1.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name panel.example.com;

    ssl_certificate /etc/ssl/example/fullchain.cer;
    ssl_certificate_key /etc/ssl/example/example.com.key;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_pass http://127.0.0.1:2053;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name sub.example.com;

    ssl_certificate /etc/ssl/example/fullchain.cer;
    ssl_certificate_key /etc/ssl/example/example.com.key;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
        proxy_pass http://127.0.0.1:10882;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name node1.example.com;

    ssl_certificate /etc/ssl/example/fullchain.cer;
    ssl_certificate_key /etc/ssl/example/example.com.key;

    location = / { return 204; }

    location ^~ /<xhttp-path-node1> {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_pass http://127.0.0.1:<xray-local-port-node1>;
    }

    location / { return 404; }
}
```

For remote node VPS-B, use the same `nodeN.example.com` server block and proxy its XHTTP path to the node-local Xray port, for example `127.0.0.1:10002`.

### Optional LinkRay Presentation Branding

If the operator wants the panel to display `LinkRay`, change only the reverse-proxy presentation layer. Do not rename the `x-ui` service, `/usr/local/x-ui`, `x-ui.db`, API paths, node tags, or subscription paths.

Nginx can rewrite bundled frontend text without rebuilding 3X-UI:

```nginx
location / {
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Accept-Encoding "";
    add_header Cache-Control "no-store, no-cache, must-revalidate" always;
    sub_filter_once off;
    sub_filter_types application/javascript text/javascript text/css application/json;
    sub_filter 3X-UI LinkRay;
    sub_filter 3x-ui LinkRay;
    sub_filter X-UI LinkRay;
    sub_filter Linkray LinkRay;
    proxy_pass http://127.0.0.1:2053;
}
```

Verify branding with a direct resource fetch, not only the browser cache:

```bash
curl --noproxy '*' --resolve panel.example.com:443:<VPS-A-IP> \
  -ks https://panel.example.com/<basePath>/assets/<login-js> |
  grep -E 'LinkRay|Linkray|3X-UI'
```

## 4. Remote Node API Access

Skip this section in single-point mode. Local inbounds on the main panel do not need a remote node registration.

In cluster mode, 3X-UI nodes are remote 3X-UI panels. The main panel polls each node's API, including `/panel/api/server/status`, with the node's API token.

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

API example for testing and adding a remote node from the main panel:

```bash
curl -fsS -X POST \
  -H "Authorization: Bearer <MAIN_API_TOKEN>" \
  -H "Content-Type: application/json" \
  "$MAIN_API/nodes/test" \
  --data-binary '{
    "scheme": "http",
    "address": "VPS-B-IP",
    "port": 2053,
    "basePath": "/<remote-base-path>/",
    "apiToken": "<REMOTE_API_TOKEN>",
    "enable": true,
    "allowPrivateAddress": false
  }'
```

If using public fallback management, verify the intended boundary:

```bash
# From VPS-A: must succeed.
curl -fsS -H "Authorization: Bearer <REMOTE_API_TOKEN>" \
  http://VPS-B-IP:2053/<remote-base-path>/panel/api/server/status

# From the operator laptop or arbitrary public source: should timeout or fail.
curl --connect-timeout 5 http://VPS-B-IP:2053/
```

## 5. Inbounds Per Node and Protocol

Single-point minimal:

```text
node1-vless-xhttp-tls
```

Cluster minimal:

```text
node1-vless-xhttp-tls
node2-vless-xhttp-tls
```

Add optional protocols only when requested:

```text
node1-trojan-tls
node1-ss-2022
node1-vless-reality
node1-trojan-reality
node1-hysteria2

# Cluster mode also adds:
node2-trojan-tls
node2-ss-2022
node2-vless-reality
node2-trojan-reality
node2-hysteria2
```

Use distinct tags/remarks. If using Cloudflare, keep protocol/transport compatibility in mind:

| Protocol | Good default | Notes |
|---|---|---|
| VLESS | XHTTP + TLS + Nginx path | Best fit for Cloudflare/CDN fronting |
| VLESS Reality | TCP + Reality direct | DNS-only direct hostname; do not orange-cloud |
| Trojan Reality | TCP + Reality direct | Supported by 3X-UI share links; DNS-only direct hostname |
| Hysteria2 | Hysteria transport + TLS direct | UDP/QUIC; must listen on UDP, not TCP |
| Trojan | TLS direct | Usually DNS-only unless fallback/SNI is designed |
| Shadowsocks 2022 | Direct port | Do not route through Cloudflare HTTP proxy |

Reality is not a blanket switch for every protocol. For "all protocols should have Reality", create Reality-capable VLESS/Trojan profiles and keep Hysteria2 and Shadowsocks as their own direct protocols.

For single-point local inbounds, create them directly on the main panel. For remote node inbounds, create or sync them from the main panel so the main database knows their `node_id` and can include them in subscriptions.

For Nginx-terminated XHTTP, the Xray inbound should listen on localhost with transport security `none`, and the subscription should advertise the public TLS host through `externalProxy` or Host overrides:

```json
{
  "listen": "127.0.0.1",
  "port": 10001,
  "protocol": "vless",
  "streamSettings": {
    "network": "xhttp",
    "security": "none",
    "xhttpSettings": {
      "path": "/xh-random",
      "host": "node1.example.com",
      "mode": "auto"
    },
    "sockopt": {
      "trustedXForwardedFor": ["127.0.0.1", "::1"],
      "acceptProxyProtocol": false
    },
    "externalProxy": [
      {
        "forceTls": "tls",
        "dest": "node1.example.com",
        "port": 443,
        "remark": "node1.example.com",
        "sni": "node1.example.com",
        "fingerprint": "chrome"
      }
    ]
  }
}
```

`trustedXForwardedFor` avoids Xray splitHTTP/XHTTP warnings when a local reverse proxy sets forwarded headers.

For VLESS/Trojan Reality, create TCP direct inbounds on DNS-only hostnames:

```json
{
  "protocol": "vless",
  "port": 9444,
  "streamSettings": {
    "network": "tcp",
    "security": "reality",
    "tcpSettings": {"acceptProxyProtocol": false, "header": {"type": "none"}},
    "realitySettings": {
      "target": "www.yahoo.com:443",
      "serverNames": ["www.yahoo.com"],
      "privateKey": "<server-private-key>",
      "shortIds": ["<8-hex-short-id>"],
      "settings": {
        "publicKey": "<server-public-key>",
        "fingerprint": "chrome",
        "spiderX": "/"
      }
    }
  },
  "share_addr_strategy": "custom",
  "share_addr": "direct1.example.com"
}
```

Generate keys on the target node:

```bash
/usr/local/x-ui/bin/xray-linux-amd64 x25519
```

Use a separate port for Trojan Reality, for example `9445/tcp`, to keep troubleshooting simple. 3X-UI can emit `trojan://...security=reality...` links when the inbound stream security is Reality.

If one node's VLESS Reality port is reachable but intermittently fails Mihomo delay tests, move only that inbound to another allowed direct TCP port such as `8443/tcp`, update both the main-panel row and the remote node's local row, then restart `x-ui` and re-fetch the subscription. Keep the Reality `sni`/`servername` aligned with the server-side `serverNames`.

Do not treat every TLS-looking site as an equally good Reality target. If clients show `Timeout` even though DNS is correct and `nc -vz <node-ip> <port>` succeeds, change only `realitySettings.target` and `realitySettings.serverNames` first, then retest. In practice, `www.yahoo.com:443`/`www.yahoo.com` is a safer default than `www.cloudflare.com:443`, `www.microsoft.com:443`, or Apple/iCloud targets for this deployment shape. Xray may explicitly warn against Apple/iCloud Reality targets, and those should not be the default even if they pass a short smoke test.

For Hysteria2, keep the protocol as `hysteria` and set version `2` in settings. The transport must be `network: "hysteria"`:

```json
{
  "protocol": "hysteria",
  "port": 8444,
  "settings": {
    "version": 2,
    "clients": [{"auth": "<auth-token>", "email": "user001", "subId": "<subId>", "enable": true}]
  },
  "streamSettings": {
    "network": "hysteria",
    "security": "tls",
    "hysteriaSettings": {
      "version": 2,
      "auth": "",
      "udpIdleTimeout": 60,
      "masquerade": {"type": ""}
    },
    "tlsSettings": {
      "serverName": "direct1.example.com",
      "alpn": ["h3"],
      "certificates": [{
        "certificateFile": "/etc/ssl/example/fullchain.cer",
        "keyFile": "/etc/ssl/example/example.com.key"
      }]
    }
  },
  "share_addr_strategy": "custom",
  "share_addr": "direct1.example.com"
}
```

After creating or editing Hysteria2, verify `ss -lunp | grep :8444`. If `8444` appears only as TCP, the inbound is not Hysteria2 even if the remark says so.

For remote node direct inbounds, verify both databases when using direct SQLite/API repair: the main panel row must keep `node_id` and `share_addr_strategy=custom`, while the remote node local row must exist and listen. Node sync may overwrite the main row back to `share_addr_strategy=node`; re-check the decoded subscription and fix the main row or the node sync payload before handoff.

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
  node1-trojan-tls
  node1-vless-reality
  node1-trojan-reality
  node1-shadowsocks-2022
  node1-hysteria2

# Cluster mode also attaches:
  node2-vless-xhttp-tls
  node2-trojan-tls
  node2-vless-reality
  node2-trojan-reality
  node2-shadowsocks-2022
  node2-hysteria2
```

Do not create unrelated subIds per protocol. That fragments the subscription and makes quota/expiry management inconsistent.

Trojan TLS and Trojan Reality are separate profiles and can coexist in the same subscription. Use direct DNS-only hostnames for both, and verify each advertised host:port before handing the subscription to a user.

3X-UI v3 subscriptions query normalized `clients` and `client_inbounds` rows, not only the legacy `settings.clients` JSON. Prefer API/UI creation so both the JSON and normalized tables are updated. If debugging an empty subscription, inspect both:

```bash
sqlite3 /etc/x-ui/x-ui.db \
  "select id,email,sub_id,enable from clients;"
sqlite3 /etc/x-ui/x-ui.db \
  "select client_id,inbound_id from client_inbounds;"
```

An enabled client with the right `sub_id` must be linked to every inbound that should appear in the subscription.

## 6.1 Enhanced Native Clash Profile

Use this when the operator wants to keep 3X-UI as the only user, traffic, and node authority, but still needs a full Mihomo profile with visible strategy groups and routing rules. This does not require Sub-Store.

Shape:

```text
client -> https://sub.example.com/clash/<subId>
  -> Nginx /clash/<subId>
    -> localhost wrapper 127.0.0.1:3012/clash/<subId>
      -> reads native 3X-UI source http://127.0.0.1:<sub-port>/clash/<subId> with Host: sub.example.com
      -> returns enhanced Clash/Mihomo YAML
```

Keep these 3X-UI settings native:

```sql
update settings set value='https://sub.example.com/sub/' where key='subURI';
update settings set value='https://sub.example.com/clash/' where key='subClashURI';
```

Do not change `subPath=/sub/` or `subClashPath=/clash/`.

The wrapper must:

- read the native 3X-UI `/clash/<subId>` YAML as its source
- preserve the native `proxies:` entries
- replace the native minimal `proxy-groups` and `rules` with the v2ray-agent-style groups and MetaCubeX `mrs` rule-providers documented below
- optionally rewrite direct node `server` values from DNS names to VPS IPs when clients use fake-ip DNS, while preserving Reality `sni`/`servername` and Hysteria2 `sni`
- forward `subscription-userinfo`, `profile-title`, `profile-update-interval`, and `profile-web-page-url`
- stay bound to `127.0.0.1`

Nginx route:

```nginx
location ~ ^/clash/([A-Za-z0-9_-]+)/?$ {
    set $linkray_subid $1;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    proxy_buffering off;
    proxy_pass http://127.0.0.1:3012/clash/$linkray_subid;
}
```

The generic `/sub/<subId>` path can remain direct 3X-UI native output.

Validate:

```bash
curl -fsSI 'https://sub.example.com/clash/<subId>' |
  grep -iE '^(subscription-userinfo|profile-title|profile-update-interval|profile-web-page-url):'

curl -fsS 'https://sub.example.com/clash/<subId>' -o /tmp/linkray-clash.yaml
grep -nE '^(mixed-port|proxies|proxy-groups|rule-providers|rules):' /tmp/linkray-clash.yaml
grep -nE '^[[:space:]]*- name: (AUTO|自动选择|故障转移|负载均衡|节点选择|流媒体|手动切换|全球代理|DNS_Proxy|Telegram|Google|YouTube|Netflix|Spotify|HBO|Bing|OpenAI|ClaudeAI|Disney|GitHub|国内媒体|本地直连|漏网之鱼)$' /tmp/linkray-clash.yaml
python3 - <<'PY'
import yaml
with open('/tmp/linkray-clash.yaml') as f:
    data = yaml.safe_load(f)
for proxy in data.get('proxies', []):
    print(proxy.get('name'), proxy.get('server'), proxy.get('sni'), proxy.get('servername'))
PY
mihomo -t -f /tmp/linkray-clash.yaml
```

Reality timeout validation should test more than a single green click in the UI. A reliable handoff test is three rounds across all proxies through Mihomo's delay API; Reality nodes should consistently return a delay rather than `503 Service Unavailable` or `Timeout`. If Hysteria2 is stable but Reality is intermittent, focus on Reality target/site choice and direct TCP port choice, not Sub-Store, rule providers, or the Clash wrapper.

For direct anti-block mode, the native 3X-UI source should already contain only:

```text
VLESS Reality
Trojan Reality
Hysteria2
```

Keep Sub-Store removed in this mode unless format conversion or multi-source subscription processing becomes necessary again.

## 6.2 Subscription Adapter with Sub-Store

Use this only when the native 3X-UI subscription imports incorrectly in Clash/Mihomo clients, for example:

```text
yaml: unmarshal errors:
  line 1: cannot unmarshal !!str ... into config.RawConfig
  line 1: cannot unmarshal !!seq into config.RawConfig
```

Those errors mean the client received the wrong top-level format for a full Clash/Mihomo profile. A raw base64 link list is a string, and a JSON share list is a sequence. A Clash/Mihomo profile importer expects a YAML mapping, usually starting with `proxies:`.

Run Sub-Store only on the main panel VPS. Remote nodes do not need Docker, Node.js, or a Sub-Store process.

Node.js direct deployment shape:

```bash
# On VPS-A only.
mkdir -p /opt/sub-store/data
git clone https://github.com/sub-store-org/Sub-Store.git /opt/sub-store/app
cd /opt/sub-store/app/backend
pnpm install --frozen-lockfile
pnpm bundle:esbuild
```

Systemd environment:

```text
SUB_STORE_BACKEND_API_HOST=127.0.0.1
SUB_STORE_BACKEND_API_PORT=3011
SUB_STORE_BACKEND_PREFIX=1
SUB_STORE_FRONTEND_BACKEND_PATH=/api-ss-<random>
SUB_STORE_DATA_BASE_PATH=/opt/sub-store/data
```

Systemd service:

```ini
[Unit]
Description=Sub-Store backend
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/sub-store/app/backend
EnvironmentFile=/etc/default/sub-store
ExecStart=/usr/local/bin/node /opt/sub-store/app/backend/sub-store.min.js
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

The `SUB_STORE_BACKEND_PREFIX=1` line is required. Without it, `SUB_STORE_FRONTEND_BACKEND_PATH` is only frontend metadata and the random public path returns 404.

Nginx pattern:

```nginx
server {
    listen 443 ssl http2;
    server_name store.example.com;

    ssl_certificate /etc/ssl/example/fullchain.cer;
    ssl_certificate_key /etc/ssl/example/example.com.key;

    location = / {
        return 302 https://sub-store.vercel.app/?api=https://store.example.com/api-ss-<random>;
    }

    location ^~ /api-ss-<random>/ {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
        proxy_pass http://127.0.0.1:3011;
    }

    location / { return 404; }
}
```

Create a full adapter subscription from the 3X-UI Clash source:

```bash
API='http://127.0.0.1:3011/api-ss-<random>'

curl -fsS -X POST \
  -H 'Content-Type: application/json' \
  --data-binary '{
    "name": "linkray-full",
    "source": "remote",
    "url": "https://sub.example.com/clash/<subId>",
    "ua": "clash-meta",
    "process": [],
    "ignoreFailedRemoteSub": true
  }' \
  "$API/api/subs"
```

Public user-facing adapted URL:

```text
https://store.example.com/api-ss-<random>/download/linkray-full?target=ClashMeta&includeUnsupportedProxy=true&prettyYaml=true
```

For automatic 3X-UI UI-generated user links, prefer a dynamic route on the existing subscription domain. This avoids creating one Sub-Store subscription item per user.

Do not return only the Sub-Store `proxies:` fragment if the target client imports complete Clash/Mihomo profiles. Some clients validate the YAML but show no usable nodes because there is no `proxy-groups` entry. Add a small localhost wrapper that fetches the dynamic Sub-Store output and returns a full profile containing `mixed-port`, `proxies`, visible `proxy-groups`, `rule-providers`, and `rules`.

Wrapper service shape:

```text
127.0.0.1:3012/store/<subId>
  -> reads 127.0.0.1:3011/api-ss-<random>/download/linkray-dynamic?...url=https://sub.example.com/clash/<subId>
  -> reads response metadata from https://sub.example.com/clash/<subId>
  -> returns full Clash/Mihomo YAML profile
```

The wrapper must stay localhost-only. It is a presentation layer, not a node service.

Forward subscription metadata headers from the native 3X-UI source to the adapted response. At minimum, forward `subscription-userinfo`; also forward `profile-title`, `profile-update-interval`, `profile-web-page-url`, and `content-disposition` when present. Clients such as FlClash use `subscription-userinfo` to display used traffic, total traffic, and expiry. If this header is dropped, the adapted profile can still import and connect, but the profile card will not show traffic quota or time.

For direct anti-block mode, filter the adapted `proxies:` list to direct-resistant protocols only:

```text
keep:
  vless + reality
  trojan + reality
  hysteria2

remove from the user-facing profile:
  vless-xhttp / xhttp over Cloudflare
  trojan-tls
  shadowsocks-2022
  any node that uses a Cloudflare orange-cloud hostname or preferred IP path
```

The original 3X-UI inbounds can be disabled after the direct profile is verified. Back up `/etc/x-ui/x-ui.db` before disabling inbounds, then restart `x-ui` and verify only the direct ports remain listening.

When routing rules are requested, the wrapper should add visible strategy groups and map `meta-rules-dat` providers to those groups. The user should see v2ray-agent-style service groups in the client proxy page, not only raw proxy nodes.

Required strategy groups for the full profile:

```text
AUTO
自动选择
故障转移
负载均衡
节点选择
流媒体
手动切换
全球代理
DNS_Proxy
Telegram
Google
YouTube
Netflix
Spotify
HBO
Bing
OpenAI
ClaudeAI
Disney
GitHub
国内媒体
本地直连
漏网之鱼
```

Example group shape:

```yaml
proxy-groups:
  - name: 自动选择
    type: url-test
    url: http://www.gstatic.com/generate_204
    interval: 300
    tolerance: 80
    proxies:
      - node1-vless-reality
      - node2-vless-xhttp
  - name: 故障转移
    type: fallback
    url: http://www.gstatic.com/generate_204
    interval: 300
    tolerance: 80
    proxies:
      - node1-vless-reality
      - node2-vless-xhttp
  - name: 负载均衡
    type: load-balance
    url: http://www.gstatic.com/generate_204
    interval: 300
    strategy: consistent-hashing
    proxies:
      - node1-vless-reality
      - node2-vless-xhttp
  - name: 节点选择
    type: select
    proxies:
      - 手动切换
      - 自动选择
      - 故障转移
      - 负载均衡
      - DIRECT
      - node1-vless-reality
      - node2-vless-xhttp
  - name: OpenAI
    type: select
    proxies:
      - 节点选择
      - 自动选择
      - 故障转移
      - node1-vless-reality
      - node2-vless-xhttp
```

Use the actual proxy names extracted from the Sub-Store `proxies:` output; the names above are placeholders.

```yaml
rule-providers:
  private:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-private.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/private.mrs"
  category-ads-all:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-category-ads-all.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/category-ads-all.mrs"
  cn:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-cn.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/cn.mrs"
  geolocation-!cn:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-geolocation-not-cn.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/geolocation-!cn.mrs"
  openai:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-openai.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/openai.mrs"
  github:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-github.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/github.mrs"
  google:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-google.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/google.mrs"
  youtube:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-youtube.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/youtube.mrs"
  netflix:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-netflix.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/netflix.mrs"
  spotify:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-spotify.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/spotify.mrs"
  hbo:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-hbo.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/hbo.mrs"
  bing:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-bing.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/bing.mrs"
  disney:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-disney.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/disney.mrs"
  anthropic:
    type: http
    behavior: domain
    format: mrs
    interval: 86400
    path: ./ruleset/geosite-anthropic.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/anthropic.mrs"
  telegram:
    type: http
    behavior: ipcidr
    format: mrs
    interval: 86400
    path: ./ruleset/geoip-telegram.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/telegram.mrs"
  cn-ip:
    type: http
    behavior: ipcidr
    format: mrs
    interval: 86400
    path: ./ruleset/geoip-cn.mrs
    url: "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geoip/cn.mrs"

rules:
  - RULE-SET,private,DIRECT
  - RULE-SET,category-ads-all,REJECT
  - RULE-SET,telegram,Telegram,no-resolve
  - RULE-SET,openai,OpenAI
  - RULE-SET,github,GitHub
  - RULE-SET,google,Google
  - RULE-SET,youtube,YouTube
  - RULE-SET,netflix,Netflix
  - RULE-SET,spotify,Spotify
  - RULE-SET,hbo,HBO
  - RULE-SET,bing,Bing
  - RULE-SET,disney,Disney
  - RULE-SET,anthropic,ClaudeAI
  - RULE-SET,geolocation-!cn,全球代理
  - RULE-SET,cn,本地直连
  - RULE-SET,cn-ip,本地直连,no-resolve
  - MATCH,漏网之鱼
```

Validate with `mihomo -t`. If the client only shows `PROXY` and `AUTO`, it is still using an old cached profile or the wrapper is returning only generic groups. Delete and re-import the subscription after changing the wrapper.

```nginx
server {
    server_name sub.example.com;

    location ~ ^/store/([A-Za-z0-9_-]+)/?$ {
        set $linkray_subid $1;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
        proxy_pass http://127.0.0.1:3012/store/$linkray_subid;
    }

    location / {
        proxy_pass http://127.0.0.1:10882;
    }
}
```

Then update only the displayed 3X-UI subscription URIs:

```sql
update settings
set value='https://sub.example.com/store/'
where key in ('subURI', 'subClashURI');
```

Do not change `subPath=/sub/`, `subClashPath=/clash/`, or the native 3X-UI `/clash/<subId>` route. Sub-Store uses the native Clash route as its source.

The UI-created user's copied subscription should then be:

```text
https://sub.example.com/store/<subId>
```

Verify it starts with a YAML mapping and includes the expected node count:

```bash
curl -fsS 'https://store.example.com/api-ss-<random>/download/linkray-full?target=ClashMeta&includeUnsupportedProxy=true&prettyYaml=true' |
  tee /tmp/linkray-full.yaml |
  awk 'NR==1 {print}'

grep -c '^[[:space:]]*name: ' /tmp/linkray-full.yaml

curl -fsS 'https://sub.example.com/store/<subId>' |
  awk 'NR==1 {print}'

curl -fsSI 'https://sub.example.com/store/<subId>' |
  grep -iE '^(subscription-userinfo|profile-title|profile-update-interval|profile-web-page-url):'

curl -fsS 'https://sub.example.com/store/<subId>' -o /tmp/linkray-store.yaml
grep -nE '^(mixed-port|proxies|proxy-groups|rules):' /tmp/linkray-store.yaml
python3 - <<'PY'
import yaml
with open('/tmp/linkray-store.yaml') as f:
    data = yaml.safe_load(f)
for proxy in data.get('proxies', []):
    print(proxy.get('name'), proxy.get('type'), proxy.get('server'), proxy.get('network'), bool(proxy.get('reality-opts')))
PY
grep -nE '^[[:space:]]*- name: (AUTO|自动选择|故障转移|负载均衡|节点选择|流媒体|手动切换|全球代理|DNS_Proxy|Telegram|Google|YouTube|Netflix|Spotify|HBO|Bing|OpenAI|ClaudeAI|Disney|GitHub|国内媒体|本地直连|漏网之鱼)$' /tmp/linkray-store.yaml
grep -nE '^(rule-providers|rules):|RULE-SET' /tmp/linkray-store.yaml
grep -c '^[[:space:]]*name: ' /tmp/linkray-store.yaml
mihomo -t -f /tmp/linkray-store.yaml

for host in ca.example.com la.example.com; do
  nc -vz -w 5 "$host" 9444
  nc -vz -w 5 "$host" 9445
done
```

## 6.3 Server Hardening

Enable BBR on every VPS:

```bash
modprobe tcp_bbr 2>/dev/null || true
cat >/etc/sysctl.d/99-cyclelink-bbr.conf <<'EOF'
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
EOF
sysctl --system
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc
lsmod | grep -E '(^tcp_bbr|bbr)' || true
```

Configure UFW only after confirming SSH access. Example for VPS-A:

```bash
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow 2222/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 9444/tcp  # VLESS Reality, when enabled
ufw allow 9445/tcp  # Trojan Reality, when enabled
ufw allow 8444/udp  # Hysteria2, when enabled
ufw --force enable
ufw status verbose
```

Example for VPS-B public fallback node API:

```bash
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow 2222/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow from <VPS-A-IP> to any port 2053 proto tcp
ufw deny 2053/tcp
ufw allow 9444/tcp  # VLESS Reality, when enabled
ufw allow 9445/tcp  # Trojan Reality, when enabled
ufw allow 8444/udp  # Hysteria2, when enabled
ufw --force enable
ufw status verbose
```

Configure Fail2Ban for the actual SSH port:

```bash
cat >/etc/fail2ban/jail.d/cyclelink-sshd.local <<'EOF'
[sshd]
enabled = true
port = 2222
backend = systemd
maxretry = 5
findtime = 10m
bantime = 1h
EOF

fail2ban-server -t
systemctl enable --now fail2ban
systemctl restart fail2ban
fail2ban-client status
fail2ban-client status sshd
```

Do not add broad Nginx Fail2Ban jails when node domains are behind Cloudflare orange-cloud unless the jail is Cloudflare-aware; otherwise bans may target Cloudflare edge IPs instead of real clients.

## 7. Verification

Before handing over the subscription:

```bash
# On every VPS
systemctl is-active x-ui
ss -tlnp | grep -E ':(443|10882|<panel-port>|9444|9445) '
ss -lunp | grep -E ':(8444) ' || true

# Cluster mode only: from VPS-A to each remote node
curl -fsS -H "Authorization: Bearer <NODE_API_TOKEN>" \
  https://<node-management-host>:<panel-port>/<base-path>/panel/api/server/status

# Public subscription smoke tests
curl -I https://sub.example.com/sub/<subId>
curl -I https://sub.example.com/clash/<subId>
curl -I https://sub.example.com/json/<subId>

# Decode the generic subscription and count links.
curl -fsS https://sub.example.com/sub/<subId> | base64 -d

# Confirm expected schemes/security combinations.
curl -fsS https://sub.example.com/sub/<subId> | base64 -d | grep -E 'security=reality|hysteria2://'

# Confirm TLS and source routing without local proxy or fake-IP DNS.
curl --noproxy '*' --resolve panel.example.com:443:<VPS-A-IP> \
  -o /dev/null -w '%{http_code} %{ssl_verify_result}\n' \
  https://panel.example.com/<basePath>/

# Confirm BBR and Fail2Ban.
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc
fail2ban-client status sshd

# Confirm acme.sh renewal is installed.
/root/.acme.sh/acme.sh --list
crontab -l | grep acme.sh
```

Then import the Clash/Mihomo subscription in a client and verify it contains every expected node/protocol remark exactly once.

Expected count example for two nodes with XHTTP, Trojan TLS, SS2022, VLESS Reality, Trojan Reality, and Hysteria2: 12 generic links total, including four Reality links and two Hysteria2 links.

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

For single-point output, omit remote node API status:

```text
Main panel: https://panel.example.com/<basePath>
Mode: single-point
Local node: node1
User: user001
Generic subscription: https://sub.example.com/sub/<subId>
Clash/Mihomo: https://sub.example.com/clash/<subId>
JSON: https://sub.example.com/json/<subId>
Included profiles:
  - node1-vless
  - node1-trojan
Security:
  - Subscription public over HTTPS
  - Panel admin access restricted
```
