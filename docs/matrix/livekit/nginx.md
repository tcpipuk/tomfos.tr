# Deploying LiveKit with Nginx (Manual Docker Setup)

This guide is for setting up LiveKit with Docker Compose yourself, specifically for integrating with
an Nginx reverse proxy you already manage. Compared to the Caddy generator, this gives you more knobs
to turn and fits better if Nginx is already part of your workflow for services like Synapse or
Continuwuity.

If you haven't already, please review the [main LiveKit deployment guide](README.md) for
prerequisites, firewall rules, and DNS configuration before proceeding. Getting those right first
will save potential headaches later.

## Configuration

1. **Create Directory:** Let's make a dedicated spot for your LiveKit config and data. Something like
    `/opt/livekit/` is a common choice: `mkdir -p /opt/livekit && cd /opt/livekit`
2. **Generate Keys:** We need API keys for LiveKit. Use the `livekit-server` Docker image itself to
    generate these. **Store the output securely**, perhaps in your password manager.

    ```bash
    docker run --rm livekit/livekit-server:latest generate-keys
    ```

    This will output an `API Key` (e.g. `APIp2ow9ec5MJQd`) and an `API Secret`
    (e.g. `CjwZBVnAeaCVebgtYyWfEjLehqjR6I5PHEkTVdUci2Q`).
3. **Create `livekit.yaml`:** Create a configuration file `/opt/livekit/livekit.yaml`. This file
    controls the LiveKit server itself. Start with this example, but pay close attention to the
    `rtc`, `turn`, and `keys` sections.

    ```yaml title="/opt/livekit/livekit.yaml"
    port: 7880
    bind_addresses: [ 0.0.0.0 ]
    rtc:
      tcp_port: 7881 # WebRTC over TCP
      port_range_start: 50000 # UDP port range for WebRTC
      port_range_end: 60000
      use_external_ip: false # Set to true if not behind NAT/proxy handling external IP
    turn:
      enabled: true
      domain: livekit.your.domain # Must match your domain
      tls_port: 443 # TURN/TLS will run on the main HTTPS port handled by Nginx
      udp_port: 3478 # TURN/UDP
      external_tls: true # Nginx handles TLS termination
    keys:
      # Replace API_KEY and API_SECRET with the values generated above
      API_KEY: API_SECRET
      # Example:
      # APIp2ow9ec5MJQd: CjwZBVnAeaCVebgtYyWfEjLehqjR6I5PHEkTVdUci2Q
    logging:
      level: info
    ```

    - Replace `livekit.your.domain` with your actual public domain.
    - Replace `API_KEY` and `API_SECRET` with the keys you generated.
    - Adjust `port_range_start` and `port_range_end` if needed, ensuring they match firewall rules.
    - `turn.external_tls: true` is crucial here. It tells LiveKit that Nginx (running externally to
      this container) will handle the TLS termination for TURN connections arriving on port 443,
      rather than LiveKit trying (and failing) to handle TLS itself when proxied this way.
    - `use_external_ip: false` is generally correct when behind a proxy like Nginx.
4. **Create `docker-compose.yaml`:** Create `/opt/livekit/docker-compose.yaml` to manage the LiveKit
    server and Redis containers.

    ```yaml title="/opt/livekit/docker-compose.yaml"
    services:
      livekit:
        command: --config /etc/livekit.yaml
        deploy:
          resources:
            limits:
              cpus: 2.0
              memory: 2G
        image: livekit/livekit-server:latest
        network_mode: host
        restart: unless-stopped
        volumes:
          - ./livekit.yaml:/etc/livekit.yaml
    ```

    - We use `network_mode: host` for optimal performance, particularly for the high-throughput UDP
      traffic used by WebRTC (ports 50000-60000) and TURN (port 3478). This bypasses potential
      overhead or limitations associated with Docker's network address translation (NAT) for these
      port ranges.
    - **Implication:** The LiveKit container now directly shares the host's network stack. It will
      bind directly to the host machine's ports specified in `livekit.yaml` (TCP 7880, 7881, UDP 3478)
      and the configured UDP range (50000-60000). Ensure these ports are not already in use by
      other services running directly on the host.
    - Nginx, running on the same host, can still proxy requests to `127.0.0.1:7880` and
      `127.0.0.1:7881` as LiveKit will be listening on the host's loopback interface.

## Nginx Configuration

Time to set up Nginx. It needs to act as the front door, handling HTTPS and routing requests to the
correct internal LiveKit ports based on the path. Create a new Nginx server block, for example, in
`/etc/nginx/sites-available/livekit.your.domain.conf`, using SSL/TLS certificates (e.g. from Let's
Encrypt via Certbot).

```nginx title="/etc/nginx/sites-available/livekit.your.domain.conf"
# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name livekit.your.domain;
    return 301 https://$host$request_uri;
}

# Main LiveKit server block
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name livekit.your.domain;

    # SSL Configuration (ensure paths are correct)
    ssl_certificate /etc/letsencrypt/live/livekit.your.domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/livekit.your.domain/privkey.pem;
    # Include other SSL settings like protocols, ciphers, HSTS as needed
    # ssl_protocols TLSv1.2 TLSv1.3;
    # ssl_ciphers '...';
    # add_header Strict-Transport-Security "max-age=63072000" always;

    # Increase max body size for potential large uploads/data
    client_max_body_size 100M;

    # Proxy WebSocket connections (LiveKit signaling)
    location / {
        proxy_pass http://127.0.0.1:7880; # Forward to LiveKit container's port 7880
        proxy_set_header Host $http_host; # Use $http_host to preserve port if present
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400s; # Allow long-lived connections
    }

    # Proxy WebRTC/TCP connections (TURN/TLS is also handled here)
    # LiveKit uses path prefixes for different protocols on the same port
    location /rtc {
        proxy_pass http://127.0.0.1:7881; # Forward to LiveKit container's port 7881
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400s;
    }

     # TURN/TLS requests also arrive here on port 443 but are handled internally by LiveKit
     # via the /rtc proxy pass, assuming `turn.external_tls: true` is set in livekit.yaml
}
```

- Replace `livekit.your.domain` throughout with your actual domain.
- Ensure the `ssl_certificate` and `ssl_certificate_key` paths are correct for your setup.
- The `proxy_set_header Upgrade` and `proxy_set_header Connection "upgrade"` directives are essential
    for WebSocket connections used by LiveKit to function correctly through the proxy.
- The long `proxy_read_timeout` helps prevent connections from being dropped during long calls.
- Enable the site (e.g. `ln -s /etc/nginx/sites-available/livekit.your.domain.conf /etc/nginx/sites-enabled/`)
    and test the Nginx configuration (`nginx -t`).
- Reload Nginx: `systemctl reload nginx`.

## Starting the Services

1. Navigate back to your LiveKit configuration directory: `cd /opt/livekit/`
2. Start the LiveKit container: `docker compose up -d`
3. Check the logs to ensure it started correctly: `docker compose logs -f`

With the services running and Nginx configured, return to the [main LiveKit deployment guide](README.md)
to configure Matrix integration and learn about upgrading and troubleshooting.
