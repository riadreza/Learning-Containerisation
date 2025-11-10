# HAProxy Docker Deployment with Let’s Encrypt (Auto SSL Renewal)

HAProxy Docker Deployment with Let’s Encrypt (Auto SSL Renewal)

## Overview

\
This setup runs HAProxy in a Docker container that:\
\- Load balances traffic between two IIS servers (192.168.9.45 and 192.168.9.46)\
\- Uses Let’s Encrypt certificates for HTTPS\
\- Supports automatic SSL renewal via Certbot\
\- Runs on the host ha.riad.com.bd (IP: X.X.X.X)\
\
\


## Server Topology

| Component    | Hostname       | IP Address    | Role                |
| ------------ | -------------- | ------------- | ------------------- |
| IIS Server 1 | riad01         | 192.1 68.9.45 | Backend 1           |
| IIS Server 2 | riad02         | 192.168.9.46  | Backend 2           |
| HAProxy      | ha.riad.com.bd | X.X.X.X       | Load Balancer + SSL |

Prerequisites\
&#x20;
------

\- Ensure Docker and Docker Compose are installed.\
\- Open ports 80 and 443 on the HAProxy host.\
\- Domain (ha.riad.com.bd) must point to the public IP of the HAProxy host (for Let’s Encrypt HTTP challenge).\
\


\
\
\


## Directory Structure

\
/opt/haproxy/\
├── docker-haproxy.yml\
├── haproxy.cfg\
└── certbot-init.sh\
\


Docker Compose File (docker-haproxy.yml)\
&#x20;
------

```yaml
services:
  haproxy:
    image: haproxy:lts-alpine  # Use lightweight Alpine-based HAProxy image
    container_name: haproxy    # Explicit container name for easy reference
    restart: unless-stopped    # Automatically restart unless manually stopped
    ports:
      - "80:80"               # HTTP traffic
      - "443:443"             # HTTPS traffic
      - "8080:8080"           # HAProxy stats dashboard for monitoring
    environment:
      - TZ=Asia/Dhaka         # Set timezone for proper logging
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro  # Mount custom config
      - ./certs:/opt/haproxy/certs:ro    # SSL certificates directory
      - /etc/localtime:/etc/localtime:ro # Sync host time
    networks:
      - haproxy-net           # Connect to custom network

  certbot:
    image: certbot/certbot:latest  # Official Certbot image for SSL certificates
    container_name: certbot
    restart: unless-stopped
    volumes:
      - ./certs:/etc/letsencrypt   # Persist Let's Encrypt certificates
      - ./certs:/opt/haproxy/certs # Share certificates with HAProxy
    entrypoint: >
      /bin/sh -c '
      trap exit TERM;  # Handle graceful shutdown
      
      # Initial certificate obtain
      certbot certonly --standalone -d ha.riad.com.bd --non-interactive --agree-tos -m riadreza41@gmail.com;
      
      # Combine private key and certificate for HAProxy
      cat /etc/letsencrypt/live/ha.riad.com.bd/privkey.pem /etc/letsencrypt/live/ha.riad.com.bd/fullchain.pem > /opt/haproxy/certs/live/ha.riad.com.bd/haproxy.pem;
      
      # Set proper permissions
      chmod 644 /opt/haproxy/certs/live/ha.riad.com.bd/haproxy.pem;
      
      # Loop to auto-renew every 12 hours
      while :; do
        sleep 12h & wait $${!};
        
        # Renew certificates
        certbot renew --deploy-hook "cat /etc/letsencrypt/live/ha.riad.com.bd/privkey.pem /etc/letsencrypt/live/ha.riad.com.bd/fullchain.pem > /opt/haproxy/certs/live/ha.riad.com.bd/haproxy.pem && chmod 644 /opt/haproxy/certs/live/ha.riad.com.bd/haproxy.pem && docker kill -s HUP haproxy";
        # --deploy-hook: After renewal, rebuild HAProxy cert and reload config
      done
      '
    networks:
      - haproxy-net

networks:
  haproxy-net:
    external: true  # Use pre-existing external network
```

### HAProxy Configuration (haproxy.cfg)

```context
# =====================
# Global Settings
# =====================
global
    # Log to stdout in raw format using local0 facility
    log stdout format raw local0
    # Run as a daemon in background
    daemon
    # Maximum concurrent connections
    maxconn 2000
    # Default DH parameter size for SSL (affects Perfect Forward Secrecy)
    tune.ssl.default-dh-param 2048
    # SSL cipher suites - only HIGH strength, exclude anonymous and MD5
    ssl-default-bind-ciphers HIGH:!aNULL:!MD5
    # Disable insecure SSL/TLS versions
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11

# =====================
# Defaults
# =====================
defaults
    # Inherit logging settings from global section
    log global
    # Default mode (HTTP vs TCP)
    mode http
    # Don't log connections with no data
    option dontlognull
    # Number of retries on failure
    retries 3
    # Timeout for connection to server
    timeout connect 5s
    # Timeout for client inactivity
    timeout client  50s
    # Timeout for server inactivity
    timeout server  50s
    # Custom log format with timing, connection info, and request details
    log-format "%t %ci:%cp [%TR] %ft %b/%s %ST %B \"%r\""

# =====================
# Frontend: HTTP (Redirect to HTTPS)
# =====================
frontend http_front
    # Listen on all interfaces port 80
    bind *:80
    mode http
    # Permanent redirect (301) all HTTP traffic to HTTPS
    redirect scheme https code 301 if !{ ssl_fc }

# =====================
# Frontend: HTTPS
# =====================
frontend https_front
    # Listen on all interfaces port 443 with SSL certificate
    # Supports HTTP/2 and HTTP/1.1 via ALPN negotiation
    bind *:443 ssl crt /opt/haproxy/certs/live/ha.riad.com.bd/haproxy.pem alpn h2,http/1.1
    mode http
    # Add X-Forwarded-For header with client IP
    option forwardfor
    # Set X-Forwarded-Proto header to help backend apps detect HTTPS
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    # Enable HSTS (HTTP Strict Transport Security) for security
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains" if { ssl_fc }
    # Send traffic to the backend server group
    default_backend iis_servers

# =====================
# Backend: IIS servers
# =====================
backend iis_servers
    mode http
    # Load balancing algorithm - distribute requests evenly
    balance roundrobin
    # Health check method - GET request to root
    option httpchk GET /
    # Allow redispatch to another server if one fails
    option redispatch
    # Health check timeout
    timeout check 5s

    # Enable cookie-based session persistence
    # SRV cookie is inserted by HAProxy, indirect mode, no cache
   
 cookie SRV insert indirect nocache

    # Define IIS servers with cookie settings
    # Each server gets a unique cookie for session persistence
    server iis01 192.168.9.45:80 check cookie iis01
    server iis02 192.168.9.46:80 check cookie iis02

# =====================
# HAProxy Stats
# =====================
listen stats
    # Listen on port 8080 for statistics
    bind *:8080
    # Enable statistics page
    stats enable
    # Statistics page URI
    stats uri /stats
    # Auto-refresh stats every 10 seconds
    stats refresh 10s
    # Authentication for stats page (username:admin, password:XXXXXXXX)
    stats auth admin:XXXXXXXX
```

### Deployment Steps

\
1\. Prepare environment\
&#x20;  `mkdir -p /opt/haproxy && cd /opt/haproxy`\
\
2\. Copy configuration files\
\
3\. Start HAProxy\
&#x20;  `docker compose -f docker-haproxy.yml up -d`\
\
4\. Generate SSL certificate\
&#x20;   `chmod +x certbot-init.sh && ./certbot-init.sh`\
\
5\. Verify setup\
&#x20;   `Access https://ha.riad.com.bd`

### Security Recommendations

\
\- Restrict access to HAProxy admin interface.\
\- Keep Docker and HAProxy images updated.\
\- Backup certificates regularly.\
\
