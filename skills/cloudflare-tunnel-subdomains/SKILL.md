---
name: cloudflare-tunnel-subdomains
description: "Use when exposing a local service at a subdomain via Cloudflare Tunnel."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [cloudflare, tunnel, cloudflared, dns, subdomains, reverse-proxy, deploy]
    category: devops
    related_skills: [docker-test-runner, service-fit-and-selection]
---

# Publish a service at `<name>.<domain>` with Cloudflare Tunnel

## When to Use

A local service (dev server, dashboard, container, SSH) should be reachable on the internet
at a hostname you own, without opening inbound ports or owning a public IP.

The pattern: the app listens on loopback, `cloudflared` dials **out** to Cloudflare, a named
tunnel routes the hostname to `http://127.0.0.1:<port>`, and DNS is a **proxied CNAME** to
`<tunnel-id>.cfargotunnel.com`. No A record, no port forwarding, no certificate to manage.

Works the same on Linux, macOS and Windows; only the way you supervise the daemon differs.

## 0. Prerequisites

- The domain's nameservers point at Cloudflare and the zone shows **Active**.
- An API token, or an interactive `cloudflared login`.

### Token scopes (API path)

Create at `https://dash.cloudflare.com/profile/api-tokens` with **Zone → DNS → Edit** and
**Account → Cloudflare Tunnel → Edit**, plus **Zone → Zone → Read** for the id lookup.
Scope them to the one zone and account. Store it where the process can read it and no one
else can — `~/.config/cloudflare/token.env` (mode `600`) or `%USERPROFILE%\.cloudflare\`,
loaded from the environment rather than the command line.

```bash
export CF_API_TOKEN=...        # never inline in a command that lands in history
export CF_ACCOUNT_ID=...
export CF_ZONE_NAME=example.com
```

Resolve the zone and account once:

```bash
ZID=$(curl -s -H "Authorization: Bearer $CF_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=$CF_ZONE_NAME" | jq -r '.result[0].id')
ACC=$(curl -s -H "Authorization: Bearer $CF_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts" | jq -r '.result[0].id')
echo "zone=$ZID account=$ACC"
```

Every API response carries `"success"` and `"errors"`. Check those, not the HTTP status
alone — Cloudflare returns 200 with `success:false` for some permission problems.

## 1. Create the tunnel

```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/$ACC/cfd_tunnel" \
  -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json" \
  --data '{"name":"<name>","config_src":"cloudflare"}' | jq -r '.result.id'
```

`config_src: "cloudflare"` means the ingress rules live in the dashboard/API (remote-managed),
which is what step 2 writes. Use `"local"` only if you intend to ship a `config.yml` with the
connector; the two are not interchangeable.

## 2. Ingress rules

Remote-managed tunnels take the whole ingress list in one PUT — it replaces the previous
config, it does not append.

```bash
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/$ACC/cfd_tunnel/$TID/configurations" \
  -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json" \
  --data '{"config":{"ingress":[
    {"hostname":"<name>.<domain>","path":"/api/*","service":"http://127.0.0.1:<port>"},
    {"hostname":"<name>.<domain>","service":"http://127.0.0.1:<port>"},
    {"service":"http_status:404"}
  ]}}'
```

Rules are evaluated top to bottom. **The last entry must be a catch-all with no `hostname`**
(`http_status:404`, or a fallback service). Omit it and the config is rejected; put it first
and every request hits it.

Service forms: `http://`, `https://` (add `"originRequest":{"noTLSVerify":true}` for a
self-signed origin), `tcp://`, `ssh://`, `unix:/path/to.sock`, `http_status:404`.
Wildcards work — `"hostname":"*.<domain>"` matches any single label, and needs a matching
wildcard DNS record.

Read it back:

```bash
curl -s -H "Authorization: Bearer $CF_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$ACC/cfd_tunnel/$TID/configurations" | jq .
```

## 3. DNS

```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$ZID/dns_records" \
  -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json" \
  --data '{"type":"CNAME","name":"<name>","content":"'$TID'.cfargotunnel.com","proxied":true}'
```

`proxied` must be `true`; a DNS-only CNAME to `cfargotunnel.com` does not resolve for
clients. An `A`/`AAAA` record for the same name shadows the CNAME — delete it first.

Interactive equivalent (no API token, uses a browser login and writes `cert.pem`):

```bash
cloudflared tunnel login
cloudflared tunnel create <name>          # prints the tunnel id + credentials json
cloudflared tunnel route dns <name> <name>.<domain>
```

## 4. Run the connector

Get the token once and keep it in a file, not in the process list:

```bash
curl -s -H "Authorization: Bearer $CF_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$ACC/cfd_tunnel/$TID/token" | jq -r '.result'
```

Supervision, per platform:

| Platform | Foreground test | Persistent |
|---|---|---|
| Linux | `cloudflared tunnel run --token-file <path>` | systemd **user** unit (`Restart=always`, `WantedBy=default.target`) with `ExecStart=cloudflared tunnel --no-autoupdate run --token-file <path>` |
| macOS | same | `cloudflared service install <token>` (launchd), or a user LaunchAgent |
| Windows | `cloudflared.exe tunnel run --token-file <path>` | `cloudflared.exe service install <token>` installs a Windows service; run the shell as Administrator once for that step |
| Docker (all) | `docker run --rm cloudflare/cloudflared:latest tunnel --no-autoupdate run --token-file /etc/cf/token` | compose service with the token file mounted read-only; `--network host` on Linux, or point the service at `host.docker.internal` on Docker Desktop |

Under Docker the connector is no longer on the host's loopback: `127.0.0.1:<port>` in the
ingress rule must become the host alias (`host.docker.internal`) or the app's container name
on a shared network.

Always pass the token via `--token-file` (or the service installer), never
`--token <value>`: the value would be visible in `ps` and in any monitoring that reads it.

## 5. Verify — three independent checks

```bash
# a) tunnel is connected
curl -s -H "Authorization: Bearer $CF_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$ACC/cfd_tunnel/$TID" | jq -r '.result.status'
# expect: healthy

# b) DNS resolves
nslookup <name>.<domain>          # works on Windows, macOS, Linux; dig is not on Windows

# c) the origin answers through the edge
curl -sS -o /dev/null -w '%{http_code} %{time_total}s\n' https://<name>.<domain>/
```

All three must pass. A healthy tunnel plus a 502 means the connector is up and the **origin**
is not listening on the configured host:port — check the app, not Cloudflare.

## 6. Teardown (in this order)

```bash
curl -s -X DELETE "https://api.cloudflare.com/client/v4/zones/$ZID/dns_records/$RECORD_ID" -H "Authorization: Bearer $CF_API_TOKEN"
curl -s -X DELETE "https://api.cloudflare.com/client/v4/accounts/$ACC/cfd_tunnel/$TID" -H "Authorization: Bearer $CF_API_TOKEN"
# then stop + disable the service/unit, and delete the token file
```

Delete the DNS record first: a CNAME pointing at a deleted tunnel returns an error page for
every request instead of a clean NXDOMAIN.

## Pitfalls

- **No catch-all** → the config PUT is rejected; catch-all first → every hostname 404s.
- **A record instead of proxied CNAME** → the hostname resolves to a Cloudflare anycast IP
  that has no tunnel behind it.
- **Error 1033** = the hostname is routed to a tunnel that has no healthy connector. Check the
  daemon, `systemctl --user status cloudflared-<name>` / `Get-Service cloudflared`.
- **Error 502/521** = connector healthy, origin not reachable at the configured address.
- **Redirect loop / `ERR_TOO_MANY_REDIRECTS`** = the app sees `http` behind a TLS-terminating
  proxy and redirects again. Trust `X-Forwarded-Proto`, or set the app's base URL/SSL mode.
- **403 for scripted requests but fine in a browser** = a WAF/bot rule on the zone; a plain
  `curl` UA is often blocked. Check Security → Events before blaming the tunnel.
- Cloudflare injects scripts (email obfuscation, Rocket Loader), so byte size of the public
  HTML differs from local output. Compare structure, not length.
- Token files stay mode `600` (`icacls` the equivalent on Windows) and out of the repo.
- Rotating a tunnel token invalidates the old one immediately — update every connector that
  uses it in the same change.
- The zone must be active before any of this. A pending zone accepts API calls and serves
  nothing.
