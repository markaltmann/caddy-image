# AGENTS.md

This repository builds and publishes a custom **Caddy** container image that bundles additional modules:
- `github.com/caddy-dns/inwx` (INWX DNS provider)
- `github.com/hslatman/caddy-crowdsec-bouncer/crowdsec` (CrowdSec bouncer)

There is **no application source code** here; the “product” is the container build, config, and GitHub Actions automation.

## Repository layout

- `Caddy/Dockerfile.caddy` — multi-stage build that runs `xcaddy build` to compile a custom `caddy` binary, then copies it into a Wolfi base image.
- `Caddy/Caddyfile` — example Caddy configuration.
- `Caddy/.env.tmpl` — env template for local configuration (exists; content not inspected here).
- `refs.txt` — versions used for labeling/build metadata and as the change trigger for the image build workflow.
- `.github/workflows/`
  - `docker-image.yml` — builds and pushes multi-arch image when `refs.txt` changes.
  - `trigger-image-build.yml` — scheduled workflow that checks upstream versions and updates `refs.txt`.
  - `osv-scanner.yml` / `scorecard.yml` — security scanning.

## Essential workflows and how to change versions

### Version source of truth

`refs.txt` is parsed by the build workflow and used to set build args/labels.

Format observed in `refs.txt`:
- `caddy-crowdsec-bouncer=vX.Y.Z`
- `caddy-dns-inwx=vX.Y.Z`
- `caddy-image=vX.Y.Z`

The build workflow `docker-image.yml` is configured to trigger on changes to `refs.txt`.

### Image build (CI)

In `.github/workflows/docker-image.yml`:
- Parses `refs.txt` into environment variables.
- Builds `./Caddy/Dockerfile.caddy` via `redhat-actions/buildah-build@v2`.
- Targets platforms: `linux/amd64,linux/arm64`.
- Pushes to `ghcr.io/${{ github.repository }}/caddy:latest`.

### Scheduled updates (CI)

In `.github/workflows/trigger-image-build.yml`:
- Uses `git ls-remote --tags` against the two Go module repos to discover latest semantic version tag.
- Pulls `docker.io/caddy:builder` and inspects env var `CADDY_VERSION` to derive the upstream Caddy version.
- Rewrites `refs.txt` (deletes prior keys, appends new ones, removes blank lines) and commits/pushes if changed.

## Local development / verification

This repo doesn’t define a canonical local test runner. The most representative local checks are:

### Build the container image locally

Use Podman or Docker to build `Caddy/Dockerfile.caddy`.

Inputs required by the Dockerfile build (as `--build-arg`):
- `CADDY_IMAGE_VERSION`
- `BUILD_DATE`
- `GITHUB_SHA`
- `CADDY_CROWDSEC_BOUNCER_VERSION`
- `CADDY_DNS_INWX_VERSION`

These values are provided in CI from `refs.txt` plus runtime metadata.

### Run the built image

The container entrypoint runs:
- `/usr/bin/caddy run --config /etc/caddy/Caddyfile --adapter caddyfile`

So, when running locally you typically bind-mount a `Caddyfile` to `/etc/caddy/Caddyfile`.

Note: For quick local testing we recommend using `podman-compose` (see `Caddy/podman-compose.yml`).

Long-term run mode (note for agents):

- Development / test: use `podman-compose` or `podman run` as needed to iterate quickly.
- Production / long-run: migrate container services to systemd-managed units (Podman quadlets / systemd units). See [README.md](file:///Users/markaltmann/Git/caddy-image/README.md#long-term--production-systemd-quadlets) for migration notes. Typical workflow:
  1. Build the image (CI or local registry).
  2. Generate a systemd unit with `podman generate systemd --new --name <container> --files` or author a quadlet under `/etc/quadlet.d/`.
  3. Enable and start the resulting systemd service(s) (`systemctl enable --now <unit>`).

Do NOT commit secrets into the repo. Keep `.env` out of version control and use a secrets manager where possible.


## Caddyfile conventions and gotchas

- `Caddy/Caddyfile` is an example configuration.
- **Secrets must not be committed**: The configuration uses environment variables (e.g., `{$INWX_PASSWORD}`) to avoid committing secrets to version control.
- **Syntax**: Ensure Caddyfile syntax complies with official module directives (e.g. block properties separated by spaces, no colons or quoted keys).

## Security / supply chain

- OSV-Scanner is configured via `.github/workflows/osv-scanner.yml` (scheduled + PR checks).
- OpenSSF Scorecard runs via `.github/workflows/scorecard.yml`.
- Dependabot config exists at `.github/dependabot.yml`.

## Common agent tasks

### Update modules included in the custom Caddy build

Edit `Caddy/Dockerfile.caddy` and adjust the `xcaddy build` `--with ...` entries.

### Update versions/labels and trigger a new image build

Edit `refs.txt` (or let the scheduled workflow update it). Any change to `refs.txt` triggers the build workflow.

### Validate CI wiring

When modifying workflows, ensure:
- `docker-image.yml` still reads versions from `refs.txt` correctly.
- The build args passed in CI match the `ARG` names in `Caddy/Dockerfile.caddy`.
