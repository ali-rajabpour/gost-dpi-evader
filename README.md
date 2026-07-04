# GOST DPI-Resistant HTTPS Proxy

> *"The internet is a right, not a privilege. Access to information should never
> be gated by borders, firewalls, or censorship. This project exists to help
> restore that right — one encrypted packet at a time."*

A lightweight encrypted proxy you deploy on [Dokploy](https://dokploy.com/) with
nothing but a domain and a few env vars. Traffic leaves your machine as ordinary
HTTPS to a real domain with a valid Let's Encrypt certificate, so to a DPI box it
is indistinguishable from you visiting a normal website.

**Author:** Ali Rajabpour Sanati — [rajabpour.com](https://rajabpour.com)
**License:** MIT — free forever, for everyone. See [LICENSE](LICENSE).

## How it works

```
client (gost)  ──HTTPS/WSS──▶  Traefik (TLS, Let's Encrypt)  ──mws──▶  gost-tunnel
   local proxy      :443            Dokploy edge, your domain      :8080   container
```

- **gost** speaks a multiplexed WebSocket (`mws`) proxy on a secret path.
- **Traefik** (built into Dokploy) terminates real TLS on your domain and only
  forwards requests that match `Host + secret path`. Everything else 404s, so
  probing the bare domain reveals nothing.
- **DPI resistance** comes from the outer layer being genuine HTTPS/WSS to a
  domain with a valid cert — no custom TLS fingerprint, no self-signed cert, no
  odd port. The multiplex (`mws`) also cuts handshakes, so it stays fast.

## Prerequisites

- A Dokploy server (with its default Traefik + `letsencrypt` resolver).
- A domain whose A record points at the Dokploy host (or at a CDN that
  proxies to the Dokploy host).
- `gost` v3 on your client machine — https://github.com/go-gost/gost/releases
  - macOS: `brew install gost`
  - Linux: download the tarball from the releases page, extract, and move
    the `gost` binary to `/usr/local/bin/`.
  - The `dokploy-network` Docker network must exist (Dokploy creates it by
    default).

## Deploy (Dokploy)

1. Create a new **Compose** service in Dokploy, source this repo.
2. Open the **Environment** tab, paste `env-example`, fill in real values:

   | Var              | What                                             |
   |------------------|--------------------------------------------------|
   | `DOMAIN`         | Domain routed to the proxy (`proxy.example.com`) |
   | `PROXY_USER`     | Proxy username                                   |
   | `PROXY_PASSWORD` | Proxy password (`openssl rand -hex 16`)          |
   | `WS_PATH`        | Secret path, leading `/` (`echo "/$(openssl rand -hex 8)"`) |
   | `CERT_RESOLVER`  | `letsencrypt` (default) or empty for a custom cert uploaded via Dokploy UI |
   | `GOST_IMAGE`     | Optional image/tag override                      |

3. Point the domain in Dokploy's domain settings to port `8080` (or let the
   labels handle it — they already do).
4. **Deploy.** If `CERT_RESOLVER=letsencrypt`, wait for the cert to issue
   (first request may lag a few seconds). If `CERT_RESOLVER` is empty, make
   sure a custom cert for `DOMAIN` is already uploaded in Dokploy's
   **Certificates** section.
5. If using a CDN (Cloudflare, etc.), enable WebSocket support in the CDN
   settings. The CDN must also proxy HTTP (port 80) to the origin so Let's
   Encrypt's ACME HTTP-01 challenge can reach Traefik for cert renewals.

## Client setup

Run gost locally to expose a plain proxy that forwards through the tunnel.

Local **HTTP** proxy on `127.0.0.1:8080`:

```bash
gost -L "http://127.0.0.1:8080" \
     -F "mwss://PROXY_USER:PROXY_PASSWORD@DOMAIN:443?path=WS_PATH"
```

Local **SOCKS5** proxy on `127.0.0.1:1080`:

```bash
gost -L "socks5://127.0.0.1:1080" \
     -F "mwss://PROXY_USER:PROXY_PASSWORD@DOMAIN:443?path=WS_PATH"
```

`mwss` = `mws` over TLS. The cert is validated against `DOMAIN`, so no insecure
flags are needed. Point your app at the local proxy:

```bash
curl -x http://127.0.0.1:8080 https://api.ipify.org   # should show the server IP
```

## Security notes

- Keep `WS_PATH` and credentials secret — they are the only thing between the
  public internet and an open proxy.
- Rotate `PROXY_PASSWORD` / `WS_PATH` if leaked; redeploy to apply.
- The container runs with `no-new-privileges` and a 128 MB memory cap.

## Updating

Bump the tag in `docker-compose.yml` (or set `GOST_IMAGE`) and redeploy. Latest
tags: https://hub.docker.com/r/gogost/gost/tags — pin a version rather than
`latest` for reproducible deploys.

## Troubleshooting

| Symptom                          | Check                                              |
|----------------------------------|----------------------------------------------------|
| 404 from the domain root         | Expected — you must hit `WS_PATH`.                 |
| Client can't connect             | `path=` matches `WS_PATH` exactly (leading `/`).   |
| TLS / cert errors                | A record resolves to Dokploy (or CDN proxies to it); cert finished issuing.|
| Auth failures                    | No `: / @ ? #` in the password (breaks the URL).   |
| WebSocket upgrade fails          | CDN WebSocket support not enabled.                 |
| Cert renewal fails               | CDN not proxying port 80 to origin (ACME HTTP-01). |
| `i/o timeout` on client          | DNS not resolving `DOMAIN`; check `/etc/resolv.conf`. |
