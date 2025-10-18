---
icon: user-beard-bolt
cover: .gitbook/assets/Gemini_Generated_Image_ao5eqzao5eqzao5e.jpg
coverY: 0
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

Change the `acme.json`file permission.

```bash
sudo chmod 600 /opt/traefik/traefik-data/acme.json
```

```bash
sudo touch acme.json
sudo chmod 600 acme.json
```

Now, create a `taefik.yml`file in that `traefik-data`directory.&#x20;

```bash
sudo touch traefik-data/traefik.yml
```

<pre class="language-yaml"><code class="lang-yaml">api:
  dashboard: true
  insecure: false

entryPoints:
  web:
    address: :80
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: :443

providers:
  docker:
    exposedByDefault: false
    network: web

certificatesResolvers:
  cloudflare:
    acme:
      email: <a data-footnote-ref href="#user-content-fn-1">your email address</a>
      storage: /acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
log:
  level: error

</code></pre>

Now run the Docker Compose file:

```bash
sudo docker compose -f docker-traefik.yml up -d
```

[^1]: 
