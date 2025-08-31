# caddy-image
Personal Caddy Container Image with INWX and Crowdsec modules included.

## Build

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
And the triggering is done via another workflow: [![Check External References](https://github.com/markaltmann/caddy-image/actions/workflows/trigger-imgage-build.yml/badge.svg)](https://github.com/markaltmann/caddy-image/actions/workflows/trigger-imgage-build.yml)

You can find the final package here: <https://github.com/markaltmann/caddy-image/pkgs/container/caddy-image%2Fcaddy>

### Conatainer Labeling

To know, which caddy and plugins are being used, I added some labels to the container image for transparency.

## Usage

There are 2 compponents you will need:
1. Using the proper container image
2. Using a proper Caddyfile configuration

## Container Image

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
