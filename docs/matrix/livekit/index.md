# Deploying LiveKit for Matrix Video Conferencing

So, you're looking to add real-time audio and video to your Matrix setup? [LiveKit](https://livekit.io/)
is an excellent open-source WebRTC project that can power features like Element Call directly from
your own infrastructure. This guide will walk you through deploying a self-hosted LiveKit server using
Docker, keeping things consistent with how we've handled Continuwuity and Synapse deployments.

Here, we'll cover deploying LiveKit using Docker for consistency with the other guides.

## How it Works: Element Call & LiveKit

Before diving into deployment, let's quickly visualise how a client like Element Call uses your
self-hosted LiveKit server. The key is the Matrix `.well-known` discovery mechanism:

```mermaid
graph TD
    subgraph User & Matrix Client (Element Call)
        A[User clicks 'Start Call'] --> B{Fetch 'your.matrix.domain/.well-known/matrix/client'};
        B --> C{Parse 'org.matrix.msc4143.rtc_foci'<br>Find 'livekit_service_url'};
        C --> D[Connect to WSS endpoint<br>wss://livekit.your.domain];
    end

    subgraph Network Path
        D --> E{Public Internet};
        E --> F[Server Firewall / Router];
    end

    subgraph LiveKit Server Infrastructure
        F --> G[Reverse Proxy (Nginx/Caddy)<br>Handles TLS & Forwards<br>TCP 443];
        G --> H[LiveKit Server Container<br>Signaling (WSS), WebRTC/TCP, TURN/TLS];
        H --> I[Redis Container<br>Internal State];
    end

    subgraph Direct Media Paths (WebRTC)
        D --> F;
        F --> J[LiveKit TURN Service<br>UDP 3478 (Direct)];
        F --> K[Direct UDP Media<br>UDP 50000-60000 (Direct)];
    end

    P2[Other Participant's Client] --- D;
    P2 --- J;
    P2 --- K;
```

- **Discovery:** Element Call reads your main Matrix domain's `.well-known` file to find the
  `livekit_service_url`.
- **Signaling:** It connects via Secure WebSockets (`wss://`) to your LiveKit domain, usually
  through your reverse proxy (Nginx/Caddy) on port 443.
- **Media:** For the actual audio/video streams (WebRTC), clients try to connect directly via UDP
  (ports 50000-60000). If firewalls block this, they use the integrated TURN service (UDP 3478 or
  TCP 443 via the proxy) as a fallback relay. LiveKit includes TURN, so **no separate TURN server
  is needed**.

## Deployment Options

When deploying LiveKit with Docker, you've basically got two main routes, mostly depending on how
you want to handle incoming connections and TLS (HTTPS):

1. **[Caddy Deployment (Official Generator)](caddy.md):** Uses the official LiveKit configuration
   generator. This bundles Caddy, which neatly handles automatic HTTPS and reverse proxying. It's
   often simpler if you don't already have a reverse proxy setup or prefer Caddy.
2. **[Manual Docker + Nginx Deployment](nginx.md):** A more manual setup using Docker Compose and
   configuring Nginx as the reverse proxy. This gives you more control and is probably the way to go
   if you already use Nginx configuration.

Choose the guide that makes the most sense for your current server setup and comfort level. Either
way, you'll need the prerequisites covered next.

## Prerequisites

Before we get started with the specific Caddy or Nginx steps, let's make sure you have the basics
in place:

- A domain name (e.g. `livekit.your.domain`) that you control. You'll point this at your LiveKit server.
- Access to manage DNS records for this domain. We need this to point the domain to your server's IP.
- A Virtual Machine (VM) or server with Docker and Docker Compose installed. Nginx is also required
  for the Nginx guide.

## Firewall Configuration

Whether you go with Caddy or Nginx, LiveKit needs certain network ports open on your server's
firewall to work properly. Think of these as specific doors LiveKit needs open to talk to clients
and handle aspects of the WebRTC connection process:

| Port(s)     | Protocol | Direction | Required By            | Notes                                                                      |
|-------------|----------|-----------|------------------------|----------------------------------------------------------------------------|
| 80          | TCP      | Inbound   | Caddy / Nginx          | HTTP access for Let's Encrypt TLS certificate validation or HTTP redirect. |
| 443         | TCP      | Inbound   | Caddy / Nginx / LiveKit| Primary HTTPS, WSS, WebRTC/TCP fallback, TURN/TLS fallback. |
| 3478        | UDP      | Inbound   | LiveKit (TURN Service) | Primary TURN/UDP relay protocol for clients behind restrictive firewalls. |
| 50000-60000 | UDP      | Inbound   | LiveKit (Media)        | Preferred range for direct WebRTC media streams (SRTP). |

*Ensure these ports are allowed through both the server's host firewall (e.g. `ufw`, `firewalld`)
and any upstream network firewalls (cloud provider security groups, router port forwarding).*

Refer to the [LiveKit firewall documentation](https://docs.livekit.io/home/self-hosting/firewall-configuration/)
for more details.

## DNS Configuration

Simple but crucial: create DNS `A` records (and `AAAA` if you use IPv6) for your LiveKit domain
(e.g. `livekit.your.domain`). These records must point to the public IP address of the server where
LiveKit will run.

Your reverse proxy (Caddy or Nginx) needs this record to be resolvable from the public internet to
successfully obtain or use TLS certificates. You can check this using `host livekit.your.domain`.
If this isn't set up correctly, Caddy won't get certificates, and Nginx won't work properly with HTTPS.

## Matrix Integration (Element Call)

Okay, so LiveKit is running and accessible at `https://livekit.your.domain`. Now you need to tell
Matrix clients (like Element Call) how to discover and use your self-hosted instance. This is done
via the `.well-known` configuration file on your *main Matrix domain*.

Modify the file served at `https://your.matrix.domain/.well-known/matrix/client` to include the
`org.matrix.msc4143.rtc_foci` key. Make sure to replace `your.matrix.domain` and
`livekit.your.domain` with your actual domains:

```json title=".well-known/matrix/client"
{
  "m.homeserver": {
    "base_url": "https://your.matrix.domain"
  },
  "org.matrix.msc4143.rtc_foci": [
    {
      "type": "livekit",
      "livekit_service_url": "https://livekit.your.domain"
    }
  ]
}
```

**Heads up:** Ensure your web server/reverse proxy serving this `.well-known` file (which is usually
the one for your main Matrix domain, *not* the LiveKit one) has the correct CORS headers (like
`Access-Control-Allow-Origin: *`) to allow clients from any origin to fetch it. Without proper CORS
headers here, Element Call (running in a browser) won't be allowed to read this configuration.

## Upgrading

Keeping LiveKit up-to-date is straightforward with Docker Compose. The recommended approach for most
self-hosters is to use the `:latest` tag, which simplifies updates.

1. **Pull the Latest Image:** Navigate to your LiveKit directory (e.g. `/opt/livekit/`) and pull the
   newest image versions:

    ```bash
    cd /opt/livekit/
    docker compose pull
    ```

    This command pulls the latest versions for all services defined in your `docker-compose.yaml`,
    including `livekit/livekit-server:latest` and `redis:7-alpine` (or whichever versions you have specified).
2. **Restart Services:** Apply the new images by recreating the containers:

    ```bash
    docker compose up -d
    ```

If you prefer to pin to specific versions (e.g. `livekit/livekit-server:v1.5.3`), you would first
edit the `image:` line in your `/opt/livekit/docker-compose.yaml` file to specify the desired
version tag, and then run the `docker compose pull && docker compose up -d` commands as above.

## Troubleshooting Tips

Here's a checklist of common issues:

- **Check the Logs:** This is usually the first place to look. Use `docker compose logs -f` in your
  LiveKit directory. Pay attention to the `livekit` container logs first, then `redis` or
  `livekit-caddy-1` if applicable. Nginx logs are typically in `/var/log/nginx/`.
- **Check Service Status (Caddy method):** `systemctl status livekit-docker` (if using the systemd
  service from cloud-init).
- **TLS Issues:**

- **Caddy:** Look for `"certificate obtained successfully"` messages in Caddy logs. If missing,
    check DNS resolution (`host livekit.your.domain`) and firewall rules (ports 80 and 443 must be reachable).
- **Nginx:** Ensure certificate paths are correct and Nginx has permission to read them. Check
    Certbot/ACME client logs for renewal issues.

- **Firewall:** Double-check those ports! Are TCP 80/443 and UDP 3478 / 50000-60000 actually open on
  the server's firewall *and* any network firewall (like your router or cloud provider's security
  group)? Use tools like `nmap` or online port checkers.
- **DNS:** Did you *really* check the DNS record points to the right IP? Use `dig livekit.your.domain`
  or `nslookup livekit.your.domain` from an external network if possible.
- **Nginx Config (if used):** Validate with `nginx -t`. Did you remember to reload Nginx
  (`systemctl reload nginx`) after making changes? Are the `proxy_set_header Upgrade` and
  `Connection` lines definitely present for the `/` and `/rtc` locations?
- **LiveKit Connection Test:** This official tool is incredibly useful. Go to
  [https://livekit.io/connection-test](https://livekit.io/connection-test) pointing to your public
  LiveKit URL (e.g. `wss://livekit.your.domain`). It will attempt various connection methods and
  report success or failure, often pinpointing firewall or proxy issues.

Refer to the official [LiveKit troubleshooting guide](https://docs.livekit.io/home/self-hosting/vm/#troubleshooting)
for more detailed steps.
