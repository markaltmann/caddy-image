# caddy-image
Personal Caddy Container Image with INWX and Crowdsec modules included.

## Build

Caddy can be extended with additional modules and configurations to meet specific needs.
In my case I want these additional 2 ones:

- INWX <https://caddyserver.com/docs/modules/dns.providers.inwx>
- Crowdsec <https://caddyserver.com/docs/modules/crowdsec>

Building with xcaddy is done via:
```sh
xcaddy build --with github.com/caddy-dns/inwx --with github.com/hslatman/caddy-crowdsec-bouncer/crowdsec
```

The build is currently being done in a Github Action [![Build Status](https://github.com/markaltmann/caddy-image/workflows/docker-image/badge.svg)](https://github.com/markaltmann/caddy-image/actions)

You can find the package here: <https://github.com/markaltmann/caddy-image/pkgs/container/caddy-image%2Fcaddy>

### Usage

In the Caddyfile attached, you are finding an example with the INWX and crowdsec module.

Caddy can reuse also for testing simple ENV files:

```sh
# testing
$ touch Caddyfile
```
