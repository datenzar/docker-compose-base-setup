# base-setup

Project for a base setup of docker based services to run a server, including

- reverse proxy (traefik)
- ssl termination (certbot)
- container management (portainer)

This setup can be used as a base for further containers which you are building. In this setup, traefik will request a wildcard certificate from letsencrypt using the dns-challenge (in this setup configured with cloudflare API).

In order to expose a dockerized service, just make sure that this service is using the same network named `traefik`. It will then be served under the subdomain according to the service name.

Assuming that your registered domain is `example.org` the following definition would expose a service at https://whoami.example.org

```yaml
version: "3"

networks:
  default:
    external:
      name: traefik

services:
  whoami:
    # A container that exposes an API to show its IP address
    image: containous/whoami
    container_name: whoami
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}

    # Optional label to protect the service with Traefik Auth, e.g.
    # labels:
    #   - traefik.http.routers.whoami.middlewares=traefik-auth
```

## Installation

- create external network via `docker network create traefik`
- create traefik certification placeholder with `touch data/acme.json`
- adjust access to that file with `chmod 600 data/acme.json`

- copy `credentials.env.sample` to `credentials.env` and enter CloudFlare account details
- adjust email (changeme@example.org) in `data/traefik.yml` and provide your own email address
- adjust following env variable to your needs:
  - `$TRAEFIK_AUTH` need to be filled with basic auth from `htpasswd -n <username>` (see https://linux.die.net/man/1/htpasswd for details). *NOTE: within the yml-config file itself you need to make sure to escape the dollar signs with double dollars ($$), whereas setting the variable in rc, you better put it in single quotes*
  - `$ROOT_URL` need to point to the root url, e.g. `example.com`
  - `$PUID` will map the container to a user ID of your choice (you can get your own UID with `echo $UID`)
  - `$PGID` will map the container to a group ID of your choice (You can get your user group with `getent group $(whoami)`
  - `$TRUSTED_PROXIES` is the proxy of traefik network `docker network inspect traefik --format='{{(index .IPAM.Config 0).Gateway}}'`
  - `$TZ` should be according to your timezone, e.g. `Europe/Berlin`
- execute `docker-compose up -d`
- go to `https://portainer.${ROOT_DOMAIN}` and set your admin password for portainer
- done!!

## Troubleshooting

### Log ACME DNS challenge

To log ACME DNS challenge output, just enable Traefik DEBUG log level in `traefik.yml`.

### Check certificate

`nmap -p 443 --script ssl-cert whoami.${ROOT_URL}`
