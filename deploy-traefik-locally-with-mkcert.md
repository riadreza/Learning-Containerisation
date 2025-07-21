---
icon: user-beard-bolt
---

# Deploy Traefik locally with mkcert

We need to install it `mkcert` locally on the server.&#x20;

```bash
apt install mkcert
mkcert status
```

Then generate the certificate for the domain we want to give the certificate to.

```bash
mkcert -key-file certs/key.pem -cert-file certs/cert.pem "local domain name/localhost"
mkcert -key-file key.pem -cert-file cert.pem "local domain name/localhost"
```

Compose file for traefik deployment. `docker-traefik.yml`&#x20;



```yaml
services:
  traefik:
    image: traefik:v3.4
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    networks:
      - web
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/traefik/traefik.yml:/etc/traefik/traefik.yml
      - /opt/traefik/certs:/etc/traefik/certs 
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik-dashboard.rule=Host('local domain name/localhost`)"
      - "traefik.http.routers.traefik-dashboard.entrypoints=websecure"
      - "traefik.http.routers.traefik-dashboard.tls=true"
      - "traefik.http.routers.traefik-dashboard.service=api@internal"
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$2y$$XXXXXXXX"
      - "traefik.http.routers.traefik-dashboard.middlewares=dashboard-auth@docker"


networks:
  web:
    external: true
    
```
