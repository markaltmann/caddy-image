# caddy-image
Personal Caddy Container Image with INWX and Crowdsec modules included.

## Usage

Caddy can be extended with additional modules and configurations to meet specific needs.
In my case I want these additional 2 ones:

- INWX <https://caddyserver.com/docs/modules/dns.providers.inwx>
- Crowdsec <https://caddyserver.com/docs/modules/crowdsec>

Building with xcaddy is done via:
```sh
xcaddy build --with github.com/caddy-dns/inwx --with github.com/hslatman/caddy-crowdsec-bouncer/crowdsec
```

### Container
