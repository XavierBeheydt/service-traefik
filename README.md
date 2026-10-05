<!-- Copyright (c) 2026 Xavier Beheydt <xavier.beheydt@gmail.com> -->

# service-traefik

A simple Docker service for Traefik, bringing easy reverse proxy and load balancing to your Dockerized apps. Perfect for dynamic routing!

HTTP → HTTPS redirection, and the dashboard on `traefik.<DOMAIN>` behind basic auth. The defaults in `.env.example` are for **local testing** with a locally generated certificate. Production uses Let's Encrypt (HTTP-01 challenge).

## Recipes

Run `just` to list them. `up`, `down`, `start`, `stop` and `ps` are the common recipes every service provides, so the parent repo can drive all services the same way.

- `just env`: create `.env` from `.env.example` if it is missing
- `just certs`: generate a local CA (`certs/ca.crt`, kept across runs) and a certificate for `DOMAIN` and `*.DOMAIN`, plus `certs/tls.yml` that makes it Traefik's default certificate
- `just up`: create `.env` and, when `TLS_CERT_RESOLVER` is empty, the local certificate if it is missing, then `docker compose up -d`
- `just down` / `just start` / `just stop` / `just ps` / `just logs`

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

## Exposing another stack

Attach it to the external `proxy` network and add labels:

```yaml
services:
  app:
    networks: [proxy]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.${DOMAIN}`)"
      - "traefik.http.routers.app.entrypoints=websecure"

networks:
  proxy:
    external: true
```

Locally, `app.docker.localhost` is covered by the wildcard certificate.
