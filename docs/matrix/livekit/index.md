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
    Client["User Client"] --> WK["Fetch .well-known<br>from Matrix Server"]
    WK --> WSS["Connect Signaling (wss://livekit.your.domain)"];
    WSS --> Proxy["Reverse Proxy<br>TCP 443<br>(Nginx/Caddy)"];
    Client -.-> Indirect["<b>Fallback:</b><br>Indirect Media"];
    Client -. "<b>Preferred:</b><br>Direct UDP<br>(50000-50100)" .-> Internet["Peers on Internet"];
    Indirect -. TURN via TLS .-> Proxy;
    Proxy --> LiveKit["LiveKit Server<br>(SFU/TURN Controller)"];
    Indirect -. TURN via UDP .-> LiveKit;
    LiveKit --> Relay["Relays Media<br>(TURN)"];
    Relay --> Internet;
```

- **Discovery:** Element Call reads your main Matrix domain's `.well-known` file to find the
  `livekit_service_url`.
- **Signaling:** It connects via Secure WebSockets (`wss://`) to your LiveKit domain, usually
  through your reverse proxy (Nginx/Caddy) on port 443.
- **Media:** For the actual audio/video streams (WebRTC), clients try to connect directly via UDP
  (ports 50000-50100). If firewalls block this, they use the integrated TURN service (UDP 3478 or
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

LiveKit requires specific network ports to be open on your server's firewall.

| Port(s)     | Protocol | Direction | Required By             | Notes                                                                      |
|-------------|----------|-----------|-------------------------|----------------------------------------------------------------------------|
| 80          | TCP      | Inbound   | Caddy / Nginx           | HTTP access for Let's Encrypt TLS certificate validation or HTTP redirect. |
| 443         | TCP      | Inbound   | Caddy / Nginx / LiveKit | Primary HTTPS, WSS, WebRTC/TCP fallback, TURN/TLS fallback.                |
| 3478        | UDP      | Inbound   | LiveKit (TURN Service)  | Primary TURN/UDP relay protocol for clients behind restrictive firewalls.  |
| 50000-50100 | UDP      | Inbound   | LiveKit (Media)         | Preferred range for direct WebRTC media streams (SRTP). See warning below. |

!!! tip

    Ensure these ports are allowed through both the server's host firewall (e.g. `ufw`, `firewalld`)
    and any upstream network firewalls (cloud provider security groups, router port forwarding).

!!! warning "Docker Performance with Large UDP Port Ranges"

    Using very large UDP port ranges (like the old default of 10,000+ ports) can cause
    **significant delays** when starting Docker containers. This is because Docker creates individual
    firewall (`iptables`) rules for each port in the range when using default networking.

    To avoid this, I recommend both of the following:
    1. **Use a smaller port range:** Configure LiveKit (in `livekit.yaml`) to use a smaller range
       (e.g. 100-200 ports like `50000-50100`) and open only those ports in your firewall. This is
       the recommended approach for most setups.
    2. **Use Host Networking:** Configure the LiveKit container to use `network_mode: host` in Docker
       Compose, which bypasses Docker's network isolation and (some of) the `iptables` issue.

    See the specific [Caddy](caddy.md) or [Nginx](nginx.md) deployment guides for details.

Refer to the [LiveKit firewall documentation](https://docs.livekit.io/home/self-hosting/firewall-configuration/)
for more general details.

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
  // ... other keys like m.identity_server might be present ...
  "org.matrix.msc4143.rtc_foci": [
    {
      "type": "livekit",
      "livekit_service_url": "https://livekit.your.domain" // Use your LiveKit domain here
    }
  ]
  // ... potentially other keys ...
}
```

!!! warning "CORS Headers Required for `.well-known`"

    Ensure your web server/reverse proxy serving the `.well-known/matrix/client` file (which is
    usually the one for your *main Matrix domain*, **not** the LiveKit domain) has the correct CORS
    headers (like `Access-Control-Allow-Origin: *`) to allow clients from any origin to fetch it.
    Without proper CORS headers, Element Call (running in a browser) won't be allowed to read this
    configuration and will fail to initiate calls using your LiveKit server.

## Upgrading

Keeping LiveKit up-to-date is straightforward with Docker Compose. The recommended approach for most
self-hosters is to use the `:latest` tag, which simplifies updates.

1. **Pull the Latest Image:** Navigate to your LiveKit configuration directory (e.g. `/opt/livekit/`)
   and pull the newest image versions:

    ```bash
    cd /opt/livekit/
    docker compose pull
    ```

    This command pulls the latest versions for all services defined in your `docker-compose.yaml`,
    including `livekit/livekit-server:latest` and potentially `redis` or `caddy`.
2. **Restart Services:** Apply the new images by recreating the containers:

    ```bash
    docker compose up -d
    ```

If you prefer to pin to specific versions (e.g. `livekit/livekit-server:v1.5.3`), you would first
edit the `image:` line in your `/opt/livekit/docker-compose.yaml` file to specify the desired
version tag, and then run the `docker compose pull && docker compose up -d` commands as above.

## Troubleshooting Tips

LiveKit has an official tool at [https://livekit.io/connection-test](https://livekit.io/connection-test)
that can help narrow down issues - enter your **wss** URL (e.g. `wss://livekit.your.domain`) and run.
It will attempt various connection methods and report success or failure, which can help narrow down
the cause of the problem.

Refer to the official [LiveKit troubleshooting guide](https://docs.livekit.io/home/self-hosting/vm/#troubleshooting)
for more details.
