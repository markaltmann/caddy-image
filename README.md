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
sudo -- sh -c 'printf "127.0.0.1 hello.altmann.it whoami.altmann.it searx.altmann.it\n" >> /etc/hosts'
```

4. Verify the smoke tests:

```sh
curl -I https://hello.altmann.it
curl -I https://whoami.altmann.it
curl -I https://searx.altmann.it
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
- Migration outline:
  1. Build and publish the image (CI / registry).
  2. On the host generate a systemd unit with Podman:

```sh
podman generate systemd --new --name caddy --files
```

  3. Alternatively author a quadlet unit under `/etc/quadlet.d/` and enable it with `systemctl enable --now`.

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

