# Deploying Continuwuity with Docker

This guide covers deploying Continuwuity using Docker and Docker Compose, with several options for
reverse proxy configurations.

1. [Container Images](#container-images)
2. [Quick Start](#quick-start)
3. [Docker Compose Deployment](#docker-compose-deployment)
   1. [TCP Port Configuration](#tcp-port-configuration)
   2. [Unix Socket Configuration](#unix-socket-configuration)
4. [Starting the Server](#starting-the-server)

## Container Images

Official Continuwuity images are available from Forgejo:

| Image                                                 | Notes                                          |
|-------------------------------------------------------|------------------------------------------------|
| forgejo.ellis.link/continuwuation/continuwuity:latest | Stable releases, recommended for production    |
| forgejo.ellis.link/continuwuation/continuwuity:main   | Latest features, suitable for personal servers |

While the `:latest` tag is recommended for production use, the `:main` tag provides access to the
latest features and fixes. The main branch undergoes significant testing before changes are merged,
making it reliable for personal use while not necessarily "stable" for production environments.

## Quick Start

The simplest way to run Continuwuity is with a basic Docker command:

```bash
docker run -d -p 6167:6167 \
    -v continuwuity_data:/var/lib/continuwuity \
    -e CONDUWUIT_SERVER_NAME="your.server.name" \
    -e CONDUWUIT_ALLOW_REGISTRATION=false \
    -e CONDUWUIT_DATABASE_PATH="/var/lib/continuwuity"
    --name continuwuity forgejo.ellis.link/continuwuation/continuwuity:main
```

However, for production deployments, we recommend using Docker Compose with a configuration file for
better maintainability.

## Docker Compose Deployment

We provide two main deployment patterns, depending on how you want to connect to your reverse proxy:

### TCP Port Configuration

This configuration exposes Continuwuity on a TCP port, suitable for when your reverse proxy is on a
different host or when using Kubernetes:

```yaml title="docker-compose.yml"
services:
  continuwuity:
    cpus: 3
    image: forgejo.ellis.link/continuwuation/continuwuity:latest
    container_name: continuwuity
    environment:
      CONDUWUIT_CONFIG: '/var/lib/continuwuity/continuwuity.toml' # Still currently in-use
      CONDUWUIT_DATABASE_PATH: "/var/lib/continuwuity"
      CONTINUWUITY_CONFIG: '/var/lib/continuwuity/continuwuity.toml' # Added for futureproofing
    mem_limit: 4G
    ports:
      - "6167:6167"
    restart: unless-stopped
    volumes:
      - ./continuwuity.toml:/var/lib/continuwuity/continuwuity.toml:ro
      - ./data:/var/lib/continuwuity
```

### Unix Socket Configuration

This configuration uses Unix sockets for improved performance when your reverse proxy is on the same
host:

```yaml title="docker-compose.yml"
services:
  continuwuity:
    cpus: 3
    image: forgejo.ellis.link/continuwuation/continuwuity:latest
    container_name: continuwuity
    environment:
      CONDUWUIT_CONFIG: '/var/lib/continuwuity/continuwuity.toml' # Still currently in-use
      CONTINUWUITY_CONFIG: '/var/lib/continuwuity/continuwuity.toml' # Added for futureproofing
      CONTINUWUITY_DATABASE_PATH: "/var/lib/continuwuity"
    mem_limit: 4G
    restart: unless-stopped
    volumes:
      - ./continuwuity.toml:/var/lib/continuwuity/continuwuity.toml:ro
      - ./data:/var/lib/continuwuity
      - /run/continuwuity:/run/continuwuity
```

For both configurations, create a configuration file in the `data` directory:

```bash
curl -o continuwuity.toml https://forgejo.ellis.link/continuwuation/continuwuity/raw/branch/main/conduwuit-example.toml
mkdir -p data
```

See the [configuration guide](config.md) for more information on configuring Continuwuity, and the
[reverse proxy guide](reverse-proxies/index.md) for more information on how to set up a reverse
proxy to handle inbound connections to the server.

## Starting the Server

Once you've chosen and configured your setup:

```bash
# Start the services
docker compose up -d

# View the logs
docker compose logs -f
```
