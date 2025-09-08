# HomeLab
Personal Caddy Container Image with INWX and Crowdsec modules included, that power various internal containers and web services.

## OpenSSF Scorecard

Like all good internet citizen it's important to me to have a good security posture. Therefore I run the OpenSSF Scorecard on all my public repositories. Especially on this repo, with the HomeLab setup it's important to achieve a good posture and automate as much as possible in the setup, build and deployment (more on that in the podman quadlets section).

[![Scorecard supply-chain security](https://github.com/markaltmann/caddy-image/actions/workflows/scorecard.yml/badge.svg)](https://github.com/markaltmann/caddy-image/actions/workflows/scorecard.yml)

## Dependabot

The same setup includes a sensible dependabot configuration, that checks for new versions of the used dependencies and also security vulnerabilities.

[![Dependabot Status](https://api.dependabot.com/badges/status?host=github&repo=markaltmann/caddy-image)](https://dependabot.com) or:

https://github.com/markaltmann/caddy-image/network/dependencies

## Crowdsec

Crowdsec is an open source and collaborative intrusion prevention system. It is designed to analyze behaviors, respond to attacks, and share signals across a crowd of users.

More information can be found here: <https://crowdsec.net/>

## Caddy

Caddy is a powerful, enterprise-ready, open source web server with automatic HTTPS written in Go. It is known for its ease of use and simple configuration.

More information can be found here: <https://caddyserver.com/>

### Build

Caddy can be extended with additional modules and configurations to meet specific needs.
In my case I want these additional 2 ones:

- INWX <https://caddyserver.com/docs/modules/dns.providers.inwx>
- Crowdsec <https://caddyserver.com/docs/modules/crowdsec>

Building `caddy` via `xcaddy` is done simply via:
```sh
xcaddy build \ 
--with github.com/caddy-dns/inwx \ 
--with github.com/hslatman/caddy-crowdsec-bouncer/crowdsec
```

The build is currently being done in a Github Action, that triggers on the changes in the refs.txt (which is also being used for container labels): [![Build Multi-Arch Caddy Image](https://github.com/markaltmann/caddy-image/actions/workflows/docker-image.yml/badge.svg)](https://github.com/markaltmann/caddy-image/actions/workflows/docker-image.yml)  
And the triggering is done via another workflow: [![Check External References](https://github.com/markaltmann/caddy-image/actions/workflows/trigger-image-build.yml/badge.svg)](https://github.com/markaltmann/caddy-image/actions/workflows/trigger-image-build.yml)

You can find the final package here: <https://github.com/markaltmann/caddy-image/pkgs/container/caddy-image%2Fcaddy>

### Container Labeling

To know, which caddy and plugins are being used, I added some labels to the container image for transparency (example in Dockerfile):

- LABEL org.opencontainers.image.title="caddy"
- LABEL org.opencontainers.image.description="Personal Caddy Container Image with INWX and Crowdsec modules included."
- LABEL org.opencontainers.image.url="ghcr.io/markaltmann/caddy-image/caddy:latest"
- LABEL org.opencontainers.image.source="https://github.com/markaltmann/caddy-imag"
- LABEL org.opencontainers.image.version="${CADDY_IMAGE_VERSION}"
- LABEL org.opencontainers.image.created="${BUILD_DATE}"
- LABEL org.opencontainers.image.revision="${GITHUB_SHA}"
- LABEL org.opencontainers.image.licenses="The Unlicense"
- LABEL org.opencontainers.image.authors="Mark Altmann <mark@altmann.it>"
- LABEL org.opencontainers.image.component.caddy-crowdsec-bouncer="${CADDY_CROWDSEC_BOUNCER_VERSION}"
- LABEL org.opencontainers.image.component.caddy-dns-inwx="${CADDY_DNS_INWX_VERSION}"
- LABEL org.opencontainers.image.component.caddy-image="${CADDY_IMAGE_VERSION}"

## Home Lab Architecture

There are 2 compponents you will need:
1. Using the proper container image (usually provided by an owner) and run it as a "non-exposed" service locally. Routing will be done via local podman service and port
2. Using a proper Caddyfile configuration (for crowdsec and caddy certificate) for the external routing

### Container Image

Use the following container image path:

```sh
podman pull ghcr.io/markaltmann/caddy-image/caddy:latest
```

### Caddyfile
In the Caddyfile attached, you are finding an example with the INWX and crowdsec module.

Caddy can reuse also for testing simple ENV files:

```sh
# testing
$ touch Caddyfile
```
