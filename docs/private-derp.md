# Remote access and a self-hosted DERP relay

[中文](private-derp.zh-CN.md)

Palmhelm on your iPhone always connects to the Bridge on your Mac. When both are on Tailscale, Tailscale tries a direct peer-to-peer connection first. When the NAT on an office or cellular network blocks that, traffic goes through a DERP relay. Tailscale's own relays are the default and usually enough. If the nearest one is far away and slow, you can run your own DERP on a server close to you.

A self-hosted DERP only improves the relayed path; it does not make direct connections more likely. Palmhelm keeps using your Mac's address. You do not point it at the relay.

## Path and security boundaries

```text
iPhone Palmhelm
    │ Bridge API and event stream, carried by Tailscale
    ├── preferred: direct peer-to-peer ─────────────┐
    └── fallback: DERP relays WireGuard-encrypted ──┤
                                                    ▼
                                               Mac Bridge
                                                    │
                                           agents in Herdr
```

- DERP is not an exit node: the rest of your phone's traffic does not go through it.
- DERP sees metadata such as where connections come from, when, and how much traffic, but not the content: Tailscale encrypts it end to end.
- The DERP TLS certificate and Palmhelm's own identity checks are separate layers. Device pairing, tokens and certificate public-key pinning over HTTPS still apply.
- Changing the relay does not require pairing again. Do not replace Palmhelm's address with the relay's public IP.
- The relay's public IP can be found through DNS. `-verify-clients` (below) limits relaying to devices in your tailnet, but it does not stop scans or DDoS.

## Before you start

These steps assume a fresh Ubuntu 24.04 x86_64 server. If the server already runs Tailscale, a website, or systemd services with the same names, check the existing setup first instead of overwriting it.

| Example | Replace with |
| --- | --- |
| `derp.example.com` | your DERP subdomain |
| `203.0.113.10` | the server's public IPv4 (this is a documentation-only address) |
| `your-phone` | your phone's name in Tailscale |
| `private-relay` | the server's node name |
| region `900` | a custom region ID that does not clash with your existing config |

Never publish your tailnet policy, state files, certificate private keys or pairing credentials.

1. Pick a server with low latency from both your Mac and your phone, and measure from each. A test from the Mac does not tell you how the phone does on cellular.
2. Add an A record for the subdomain pointing at the server's public IPv4.
3. Check the system, memory, ports in use and the firewall:

```bash
cat /etc/os-release
uname -m
free -h
sudo ss -lntup
sudo ufw status verbose
getent ahostsv4 derp.example.com
```

4. In the cloud firewall, allow TCP 80, TCP 443 and UDP 3478. Port 80 is used to obtain and renew the certificate.
5. If the system firewall is on, allow the same three ports and keep your existing SSH rule. Do not open all ports or flush the rules.

`ss` showing a listening port does not mean it is reachable from the internet; check the cloud firewall separately.

## Install matching tailscale, tailscaled and derper

With `-verify-clients`, derper and the local tailscaled must be the same version. The commands below pin `v1.104.4`, the version this guide was tested with. Upgrade all three together later.

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl golang-go
mkdir -p "$HOME/derp-build/bin"

TS_VERSION='v1.104.4'
env GOTOOLCHAIN=auto GOMAXPROCS=2 GOFLAGS='-p=1' GOBIN="$HOME/derp-build/bin" \
  go install \
  "tailscale.com/cmd/tailscale@$TS_VERSION" \
  "tailscale.com/cmd/tailscaled@$TS_VERSION" \
  "tailscale.com/cmd/derper@$TS_VERSION"
```

`GOTOOLCHAIN=auto` downloads the Go version the modules need; the concurrency limits suit small servers. In mainland China you can add `GOPROXY=https://goproxy.cn,direct`; module checksums are still verified.

```bash
"$HOME/derp-build/bin/tailscale" version
"$HOME/derp-build/bin/derper" -version

sudo install -m 0755 "$HOME/derp-build/bin/tailscale" /usr/local/bin/tailscale
sudo install -m 0755 "$HOME/derp-build/bin/tailscaled" /usr/local/sbin/tailscaled
sudo install -m 0755 "$HOME/derp-build/bin/derper" /usr/local/bin/derper
```

When installed from source, `ERR-BuildInfo` in the version output does not mean the build failed.

## Join the server to your tailnet

Use this service file only on a server that has no tailscaled service yet:

```bash
sudo tee /etc/systemd/system/tailscaled.service > /dev/null <<'EOF'
[Unit]
Description=Tailscale node agent
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/tailscaled --state=/var/lib/tailscale/tailscaled.state --socket=/run/tailscale/tailscaled.sock --port=41641
Restart=on-failure
RestartSec=5
RuntimeDirectory=tailscale
StateDirectory=tailscale
StateDirectoryMode=0700
UMask=0077

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now tailscaled
sudo systemctl is-active tailscaled   # should print active

sudo /usr/local/bin/tailscale up --hostname=private-relay --accept-dns=false --accept-routes=false
```

Open the login link it prints and join the tailnet your Mac and iPhone are on; if device approval is enabled, approve it in the admin console. Do not share the login link.

```bash
/usr/local/bin/tailscale status
```

The server must be able to see the devices that will use the relay. If you use tags or a tightened access policy, check that they can see each other.

## Start DERP with client verification

```bash
id -u derper >/dev/null 2>&1 || sudo useradd --system --home-dir /var/lib/derper --shell /usr/sbin/nologin derper
sudo -u derper /usr/local/bin/tailscale status
```

If the `derper` user cannot list devices, fix the account or socket permissions. Do not drop `-verify-clients`.

Replace `derp.example.com` with your domain:

```bash
sudo tee /etc/systemd/system/derper.service > /dev/null <<'EOF'
[Unit]
Description=Private Tailscale DERP relay
Wants=network-online.target
Requires=tailscaled.service
After=network-online.target tailscaled.service

[Service]
Type=simple
User=derper
Group=derper
StateDirectory=derper
StateDirectoryMode=0700
WorkingDirectory=/var/lib/derper
ExecStart=/usr/local/bin/derper -hostname=derp.example.com -a=:443 -http-port=80 -stun-port=3478 -certmode=letsencrypt -certdir=/var/lib/derper/certs -c=/var/lib/derper/derper.key -verify-clients -socket=/run/tailscale/tailscaled.sock -home=blank
Restart=on-failure
RestartSec=5
UMask=0077
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now derper
sudo systemctl is-active derper
sudo ss -lntup | grep -E ':(80|443|3478)\b'
```

The certificate and the DERP node key live in `/var/lib/derper`; keep them private. `-verify-clients` admits nodes by the identity the local tailscaled sees; it is not an IP allowlist.

## Check the certificate and public endpoint

On the server:

```bash
curl --fail --show-error --silent --max-time 60 \
  --resolve derp.example.com:443:127.0.0.1 \
  --output /dev/null --write-out 'HTTP %{http_code}\n' \
  https://derp.example.com/derp/probe
```

Expect `HTTP 200`; the first request may trigger the certificate request. Do not use `curl -k` to hide certificate errors.

Run it again on your Mac without `--resolve` to confirm port 443 is reachable from the internet. On your iPhone, turn off Wi‑Fi and open `https://derp.example.com/robots.txt` in Safari; it should show `User-agent: *` and `Disallow: /`.

These checks only cover the HTTPS endpoint. They do not show that STUN, relaying or the path from phone to Mac work.

## Add your region to Tailscale

In the access-control JSON editor of the Tailscale admin console, merge this `derpMap` into the top level. **Keep your existing grants, ACLs and SSH settings; do not replace the whole policy.** If you already have a `derpMap`, merge the region into it. If `900` is taken, use another ID and change `RegionID` to match.

```json
"derpMap": {
  "OmitDefaultRegions": false,
  "Regions": {
    "900": {
      "RegionID": 900,
      "RegionCode": "private-relay",
      "RegionName": "Private DERP",
      "Nodes": [
        {
          "Name": "private-relay-1",
          "RegionID": 900,
          "HostName": "derp.example.com",
          "IPv4": "203.0.113.10",
          "DERPPort": 443,
          "STUNPort": 3478
        }
      ]
    }
  }
}
```

`OmitDefaultRegions: false` keeps Tailscale's relays as a fallback. If the server has no public IPv6, do not add one. Clients pick a relay from their own measurements. Do not put DERP behind an ordinary HTTP reverse proxy or CDN.

## Check each layer

With the phone on Tailscale and on cellular, run on your Mac:

```bash
tailscale netcheck
tailscale ping --until-direct=false --c=5 --timeout=3s your-phone
tailscale ping --tsmp --until-direct=false --c=3 --timeout=3s your-phone
```

| You see | It means |
| --- | --- |
| netcheck lists your region with a latency | the config arrived and the relay answers probes; not yet proof of relaying |
| pongs `via DERP(private-relay)` | Tailscale-level probes went through your relay |
| TSMP replies | the WireGuard data path works |
| Palmhelm on iPhone shows connected | the app connection is up; still try sending a message, status updates and reconnecting |

By default `tailscale ping` waits for a direct connection. `direct connection not established` after DERP pongs does not mean the relay failed.

## Troubleshooting

| Symptom | Next step |
| --- | --- |
| subdomain does not resolve | check the DNS record's spelling; tell "not saved yet" apart from negative caching |
| HTTP 200 on the server, handshake fails from outside | check the cloud firewall and the system firewall |
| Safari on the phone works, `tailscale ping` times out | HTTPS working does not mean DERP connections work; check device online status, region config, service logs and node visibility |
| `not authorized` or `rejected` in the log | match the device and time, then check `-verify-clients` and access-policy visibility |
| `failed to query local tailscaled` | check tailscaled, the socket, the `derper` user's permissions and that versions match |
| `missing server name`, `HTTP request to an HTTPS server` | a request without TLS SNI, or plain HTTP to port 443; this line alone does not mean your device was rejected |

```bash
sudo systemctl status tailscaled derper --no-pager
sudo -u derper /usr/local/bin/tailscale status
sudo journalctl -u derper --since "10 minutes ago" --no-pager -n 100
```

When asking for help, share times, error types and anonymised node names, not full logs.

### Proxy apps on the Mac

- This guide was tested with a proxy app's TUN mode on the whole time; you do not need to turn it off to use the relay.
- `DIRECT` in a proxy rule only chooses the exit; it does not mean traffic bypasses TUN, or that DNS returns local answers.
- If you need exclusions, Tailscale uses `100.64.0.0/10` for IPv4 and `fd7a:115c:a1e0::/48` for IPv6. The IPv4 range is shared CGNAT space, so excluding it may affect other tools that use it.
- A direct rule for UDP destination port 41641 alone does not cover every NAT-mapped port, and it does not open inbound connections on your router.

## Maintenance and rollback

Upgrade derper and tailscaled together and note the version and checksums. Watch certificate renewal, service status and traffic. A low-bandwidth server is fine for text and status updates, not for large file transfers.

To roll back, first remove only your region from the Tailscale access policy, keeping Tailscale's relays and the rest of the policy. Once your devices connect again, stop the service:

```bash
sudo systemctl disable --now derper
```

Then remove the firewall rules you added if you no longer need them. Do not flush the firewall, remove SSH rules, or stop tailscaled before checking what else uses it.

## References

- [Tailscale derper README](https://github.com/tailscale/tailscale/tree/main/cmd/derper#derp)
- [Tailscale DERP servers](https://tailscale.com/kb/1232/derp-servers)

Check flags and policy format against the official docs for the version you use.
