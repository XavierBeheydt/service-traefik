# service-traefik

A simple Docker service for Traefik, bringing easy reverse proxy and load balancing to your Dockerized apps. Perfect for dynamic routing!

HTTP → HTTPS redirection, and the dashboard on `traefik.<DOMAIN>` behind basic auth. The defaults in `.env.example` are for **local testing** with a locally generated certificate. Production uses Let's Encrypt (HTTP-01 challenge).

## Recipes

Run `just` to list them. `up`, `down`, `start`, `stop` and `ps` are the common recipes every service provides, so the parent repo can drive all services the same way.

- `just env`: create `.env` from `.env.example` if it is missing
- `just certs`: generate a local CA (`certs/ca.crt`, kept across runs) and a certificate for `DOMAIN` and `*.DOMAIN`, plus `certs/tls.yml` that makes it Traefik's default certificate
- `just networks`: create the shared `ingress` network (internal) if it is missing. It fails if `ingress` exists but is not internal (see [Networks](#networks)).
- `just up`: create `.env`, the `ingress` network and, when `TLS_CERT_RESOLVER` is empty, the local certificate if it is missing, then `docker compose up -d`
- `just down` / `just start` / `just stop` / `just ps` / `just logs`
- `just clean`: after a confirmation, remove the containers, the `letsencrypt` volume, `.env` and the generated certificates. The shared `ingress` network is kept. On the VPS this drops the Let's Encrypt certificates, which then have to be issued again (mind the rate limits).

## Local testing

```bash
just up
curl --cacert certs/ca.crt -u admin:admin https://traefik.docker.localhost/dashboard/
```

`*.docker.localhost` resolves to `127.0.0.1` without any DNS setup. The dashboard login is `admin` / `admin`. Import `certs/ca.crt` as a trusted authority in your browser to get rid of the certificate warning. If ports 80 or 443 are already taken, change `HTTP_PORT` and `HTTPS_PORT` in `.env`. Run `just certs` again after changing `DOMAIN`.

## Production

The DNS records for `<DOMAIN>` and `traefik.<DOMAIN>` must point to the VPS.

```bash
just env
htpasswd -nB admin   # paste the result into TRAEFIK_DASHBOARD_USERS (from apache2-utils)
nvim .env            # set DOMAIN, ACME_EMAIL, TRAEFIK_DASHBOARD_USERS and TLS_CERT_RESOLVER=le
just up
just logs
```

With `TLS_CERT_RESOLVER=le`, every HTTPS router gets a Let's Encrypt certificate. No local certificate is generated, and `certs/` stays empty.

## Networks

- `ingress`: shared between the stacks and **internal**, so it has no Internet access. It only carries the traffic between Traefik and the routed containers. It is not owned by any stack: the `up` recipe of every stack that uses it creates it if it is missing (`docker network create --internal ingress`).
- `egress`: Traefik's own network, with outbound access. It carries the published ports 80/443 and the calls to Let's Encrypt.

A container reaches the Internet only through a non-internal network. Each stack declares its own (`egress`) for the containers that need it, instead of getting it through `ingress` as a side effect.

`ingress` replaces `proxy`, a normal network that this stack used to own. To migrate, stop every stack (`just down` in the parent repo), run `docker network rm proxy` if it is still there, then start them again.

Docker Swarm creates its own network named `ingress` when it is initialized. This setup uses plain Compose, so there is no conflict, but enabling Swarm on the same host would require renaming this network.

## Exposing another stack

Attach the container to the external `ingress` network, add labels, and create the network in the stack's `up` recipe like `just networks` does here:

```yaml
services:
  app:
    networks:
      - ingress
      - egress   # only if the app needs to reach the Internet
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.${DOMAIN}`)"
      - "traefik.http.routers.app.entrypoints=websecure"

networks:
  ingress:
    external: true
  egress:
```

Locally, `app.docker.localhost` is covered by the wildcard certificate.
