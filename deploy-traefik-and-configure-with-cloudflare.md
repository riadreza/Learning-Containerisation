---
icon: user-beard-bolt
---

# Deploy Traefik and configure with Cloudflare

Compose file for traefik deployment. `docker-traefik.yml`&#x20;

{% code overflow="wrap" %}
```yaml
services:
  traefik:
    image: traefik:v3.4
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - 80:80
      - 443:443
    environment:
      - TZ=Asia/Dhaka
      - CF_API_EMAIL="cloudflare mail ID"
      - CF_DNS_API_TOKEN="genetate it from cloudflare"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik-data/acme.json:/acme.json
      - ./traefik-data/traefik.yml:/etc/traefik/traefik.yml
    networks:
      - web
    labels:
      - "traefik.enable=true"
      # Dashboard
      - "traefik.http.routers.dashboard.rule=Host(`traefik.your-domain.com`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=cloudflare"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth"
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$2y$$XXXXXXXXXXXXXXX"

networks:
  web:
    external: true
```
{% endcode %}

Create a directory for Traefik data.

```bash
sudo mkdir traefik-data
```

Then create a .json file for acme configuration.

```bash
sudo touch traefik-data/acme.json
```

change the `acme.json` file permission.

```bash
sudo chmod 600 /opt/traefik/traefik-data/acme.json
```

```
sudo touch acme.json
sudo chmod 600 acme.json
```

Now create a `taefik.yml` file in that `traefik` directory.&#x20;
