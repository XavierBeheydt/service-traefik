<!-- Copyright (c) 2026 Xavier Beheydt <xavier.beheydt@gmail.com> -->

# service-traefik

A simple Docker service for Traefik, bringing easy reverse proxy and load balancing to your Dockerized apps. Perfect for dynamic routing!

HTTPS with Let's Encrypt (HTTP-01 challenge), HTTP → HTTPS redirection, and the dashboard on `traefik.<DOMAIN>` behind basic auth. The DNS records for `<DOMAIN>` and `traefik.<DOMAIN>` must point to the VPS.

```bash
cp .env.example .env
htpasswd -nB admin   # paste the result into TRAEFIK_DASHBOARD_USERS (from apache2-utils)
nvim .env
docker compose up -d
docker compose logs -f traefik
```

To expose another stack, attach it to the external `proxy` network and add labels:

```yaml
services:
  app:
    networks: [proxy]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.example.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"

networks:
  proxy:
    external: true
```
