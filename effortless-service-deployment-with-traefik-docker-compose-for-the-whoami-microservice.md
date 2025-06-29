---
icon: globe-pointer
cover: .gitbook/assets/Gemini_Generated_Image_76q5i76q5i76q5i7.jpg
coverY: 0
---

# Effortless Service Deployment with Traefik: Docker Compose for the ‘whoami’ Microservice

After deploying the Traefik proxy server, we can deploy a microservice to verify that the entire system is functioning properly.

Create a Docker Compose file `docker-whoami.yml`.

```yaml
services:
  whoami:
    image: traefik/whoami:v1.10.1
    container_name: whoami
    restart: unless-stopped
    networks:
      - web
    labels:
      - "traefik.enable=true"
      # HTTP to HTTPS redirect
      - "traefik.http.routers.whoami-http.rule=Host(`whoami.your-domain`)"
      - "traefik.http.routers.whoami-http.entrypoints=web"
      - "traefik.http.routers.whoami-http.middlewares=https-redirect"
      # HTTPS configuration
      - "traefik.http.routers.whoami.rule=Host(`whoami.your-domain`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=cloudflare"

networks:
  web:
    external: true
```

Then run the compose file.

```bash
sudo docker compose -f docker-whoami.yml up -d
```

Check the service is working by running the command below:

```bash
curl -I https://whoami.your-domain
```
