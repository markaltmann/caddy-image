# HomeLab — Caddy container image

Personal Caddy container image that bundles a couple of extra modules (INWX DNS provider and CrowdSec bouncer) and example configuration to run privacy-minded, self-hosted services.

[![Scorecard supply-chain security](https://github.com/markaltmann/caddy-image/actions/workflows/scorecard.yml/badge.svg)](https://github.com/markaltmann/caddy-image/actions/workflows/scorecard.yml) [![Build Multi-Arch Caddy Image](https://github.com/markaltmann/caddy-image/actions/workflows/docker-image.yml/badge.svg)](https://github.com/markaltmann/caddy-image/actions/workflows/docker-image.yml)

Quick links
- Repository image: `ghcr.io/markaltmann/caddy-image/caddy:latest`
- Build CI: `.github/workflows/docker-image.yml`
- Version source: `refs.txt`

Table of contents
- Overview
- Quickstart (podman-compose)
- Manual build & run (podman)
- Configuration layout
- Long-term / production (systemd quadlets)
- Security notes & TODO
- Troubleshooting

Overview

This repository produces a custom `caddy` binary (via `xcaddy`) and packages it into a container image. It is intended as the reverse-proxy / TLS edge for a homelab of self-hosted services. There is no application source code here — the product is the container build, configuration, and CI that builds and publishes the image.

What this repo contains (high level)
- `Caddy/Dockerfile.caddy` — multi-stage build using `xcaddy` to compile Caddy with the selected modules.
- `Caddy/Caddyfile` — global Caddy settings (ACME via INWX, CrowdSec provider) and `import sites/*`.
- `Caddy/sites/` — per-site small Caddyfile fragments (example site files included).
- `Caddy/podman-compose.yml` — local dev compose definition for quick testing.
- `Caddy/.env.tmpl` — template for required environment variables (copy to `Caddy/.env` for local testing).
- `refs.txt` — version references used by CI to label and build the image.

Quickstart — podman-compose (fast local iteration)

1. Copy the env template and fill secrets locally (do not commit them):

```sh
cp Caddy/.env.tmpl Caddy/.env
# edit Caddy/.env and set CROWDSEC_* and INWX_* values as needed
```

2. From the `Caddy/` directory, start the compose stack (builds the image):

```sh
cd Caddy
podman-compose up --build -d
```

3. (Optional) For local smoke testing add host entries on your workstation:

```sh
sudo -- sh -c 'printf "127.0.0.1 whoami.altmann.it\n" >> /etc/hosts'
```

4. Verify the smoke tests:

```sh
curl -I https://whoami.altmann.it
```

Manual build & run (podman)

Build the image locally:

```sh
podman build -f Caddy/Dockerfile.caddy -t caddy-custom:local Caddy
```

Run the container for quick local testing (bind mounts mirror `podman-compose`):

```sh
podman run --rm \
  -p 80:80 -p 443:443 -p 443:443/udp \
  --env-file Caddy/.env \
  -v "$PWD/Caddy/Caddyfile":/etc/caddy/Caddyfile:ro \
  -v "$PWD/Caddy/data":/data \
  -v "$PWD/Caddy/config":/config \
  caddy-custom:local run --config /etc/caddy/Caddyfile --adapter caddyfile
```

Configuration layout

- The root `Caddy/Caddyfile` contains global settings (ACME via `acme_dns inwx`, and the `crowdsec` provider) and imports `Caddy/sites/*`.
- Add one site fragment per host under `Caddy/sites/` (examples: `hello.conf`, `whoami.conf`, `searxng.conf`).
- CrowdSec is enabled by default (global provider + per-site `crowdsec` middleware lines in the examples). If you need to exclude a site from CrowdSec for testing you can remove the `crowdsec` line from that site's fragment.
- TLS issuance is configured globally using the INWX DNS provider. Leave TLS implicit unless you require site-specific overrides.

Environment / secrets

- Use `Caddy/.env.tmpl` as the template. Copy to `Caddy/.env` and fill values locally. Do NOT commit `.env`.
- For local testing without real DNS/TLS you can temporarily add `tls internal` to a site fragment to use Caddy's internal CA.

Build & CI

- `refs.txt` is the source of truth for module/image versions used by the CI workflow.
- CI (`.github/workflows/docker-image.yml`) parses `refs.txt` and builds/pushes the multi-arch container image.
- The `Dockerfile.caddy` adds informative labels for provenance and included module versions.

Long-term / production (systemd quadlets)

- For production, the recommended approach is to run containers as systemd-managed units (Podman quadlets) instead of `podman-compose`.
- Ready-to-use quadlet files live in `Caddy/quadlets/`:
  - `caddynet.network` — the `caddynet` bridge network (172.19.0.0/16) that Caddy and all upstream services join. The bridge interface is named `caddynet0` so UFW rules can target it deterministically.
  - `caddy.container` — the Caddy container (uses the published GHCR image, mounts config from `/etc/caddy/`).
  - `whoami.container` — test upstream for smoke tests. Joins `caddynet` and publishes no ports; it is only reachable through the Caddy reverse proxy, so there is no way around TLS.
  - `.env.tmpl` — template for the secrets env file.
- Deployment on the server (as root):

```sh
cp Caddy/quadlets/caddynet.network Caddy/quadlets/caddy.container Caddy/quadlets/whoami.container /etc/quadlet.d/
cp Caddy/quadlets/.env.tmpl /etc/caddy/.env   # then fill in INWX_* and CROWDSEC_* values
systemctl daemon-reload
systemctl enable --now caddynet.service
systemctl enable --now caddy.service
```

- Notes:
  - Quadlet naming rule: `caddynet.network` is loaded as `caddynet.service`, and is referenced as `Network=caddynet.network` inside the `.container` file — this must match `Network=<filename>` exactly, not the service name.
  - Every upstream service that a site fragment reverse-proxies (e.g. `whoami:80`) must join the same network (`Network=caddynet.network` in its own `.container` file) for DNS name resolution to work.
  - `Restart=always` in `[Service]` overrides the container-level `Restart=` (systemd owns the restart policy for quadlets).
  - The Caddyfile reads variables (`{$INWX_USERNAME}` etc.) from the container environment, which comes from `EnvironmentFile=/etc/caddy/.env`.
- CrowdSec integration (agent on host, bouncer in container):
  - The CrowdSec agent/LAPI runs on the host by default on `127.0.0.1:8080`, which the container cannot reach. Change `api_server.listen_uri` to `0.0.0.0:8080` in `/etc/crowdsec/config.yaml` (or bind it to the bridge IP), then restrict access with UFW:

```sh
ufw allow in on caddynet0 from 172.19.0.0/16 to any port 8080 proto tcp
```

  - Register the bouncer on the host and put the key in `/etc/caddy/.env`:

```sh
cscli bouncers add caddy
```

  - Inside the container, the podman host is the bridge gateway `172.19.0.1`, so `CROWDSEC_API_URL=http://172.19.0.1:8080/`. (Alternative: `host.containers.internal`, which podman also maps to the gateway on a rootful bridge network.)
  - Keep `enable_hard_fails` commented out while bringing this up so Caddy starts even when the LAPI isn't reachable yet; the bouncer then streams decisions every `ticker_interval` (60s by default) once connectivity works.
- QUIC UDP buffers (Caddy serves HTTP/3 on 443/udp; the kernel defaults are too small). Add to `/etc/sysctl.conf` on the host and reboot or `sysctl -p`:

```sh
net.core.rmem_max=2500000
net.core.wmem_max=2500000
```

### Verification checklist (fresh server, first bring-up)

Run these on the host, in order. Assumes only the quadlets above are deployed and the
CrowdSec agent is already installed on the host (from the "Crowdsec für SSH etc." work).

1. Deploy and start, in dependency order:

```sh
mkdir -p /etc/caddy/sites
cp Caddy/Caddyfile /etc/caddy/Caddyfile
cp Caddy/sites/whoami.conf /etc/caddy/sites/   # trim to the sites you actually run
cp Caddy/quadlets/.env.tmpl /etc/caddy/.env && vi /etc/caddy/.env

systemctl daemon-reload
systemctl enable --now caddynet.service
systemctl enable --now whoami.service
systemctl enable --now caddy.service
```

2. DNS at INWX: create an A record `whoami.altmann.it → <server public IP>`. Under the
   global `acme_dns inwx` config, issuance uses the DNS-01 challenge, so TLS works
   **without** port 80 open; but port 80/443 must be open for serving traffic
   (`ufw allow 80/tcp && ufw allow 443/tcp && ufw allow 443/udp`).

3. Config sanity check before diving into logs:

```sh
podman exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
podman exec caddy caddy list-modules | grep -E "inwx|crowdsec"   # modules built in
```

4. TLS via INWX — watch issuance in the logs and on disk:

```sh
journalctl -u caddy -f | grep -iE "acme|certificate"
podman exec caddy ls /data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/
```

5. whoami resolution over the real path (from your workstation, not the server):

```sh
curl -I https://whoami.altmann.it   # expect HTTP 200 + whoami JSON body / headers
```

6. CrowdSec bouncer connectivity (container → host LAPI):

```sh
cscli bouncers list                  # 'caddy' row: "Last pull" must advance (~60s ticker)
podman exec caddy caddy crowdsec ping
podman exec caddy caddy crowdsec check <some-ip>
```

7. End-to-end ban test (proves the bouncer actually blocks). Ban your own workstation IP
   temporarily, expect an HTTP 403, then remove the decision:

```sh
cscli decisions add --ip <your-home-ip> -d 10m
curl -I https://whoami.altmann.it    # expect HTTP 403 Forbidden
cscli decisions remove --ip <your-home-ip>
```

8. Optional LAPI hardening check after step 6 passes: from an outside network confirm
   port 8080 is not reachable (`nc -zv <server-ip> 8080` should fail).

Security notes & TODO

- DO NOT commit secrets. Keep `.env` local or use a secrets manager.
- CrowdSec is used as the primary protective middleware by default for publicly available sites.
- TODO: Evaluate adding a WAF layer (Caddy + Coraza + OWASP ModSecurity Core Rule Set) as a complement/alternative to CrowdSec — run an engineering spike to evaluate false positives and operational cost.

Troubleshooting

- If Caddy fails to obtain certificates, ensure INWX credentials in `Caddy/.env` are correct and that the DNS provider supports the target zones.
- Check container logs with `podman logs <container>` and inspect files under `Caddy/config` (bind-mounted by `podman-compose.yml`).

Contributing

- If you change the modules built into the Caddy binary, update `Dockerfile.caddy` and `refs.txt` accordingly and ensure CI labels are updated.
- Keep secrets out of the repo. Use `Caddy/.env.tmpl` for examples only.

Useful links
- Caddy: https://caddyserver.com/
- CrowdSec: https://crowdsec.net/
- INWX provider docs: https://caddyserver.com/docs/modules/dns.providers.inwx
- caddy-crowdsec-bouncer: https://github.com/hslatman/caddy-crowdsec-bouncer

