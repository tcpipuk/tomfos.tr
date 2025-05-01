# Deploying LiveKit with Caddy (Official Generator)

This guide walks through deploying LiveKit using its official configuration generator. The nice thing
about this method is that it bundles Caddy, which takes care of getting HTTPS certificates and acting
as the reverse proxy automatically. It's a pretty streamlined approach, based on the
[official LiveKit VM deployment docs](https://docs.livekit.io/home/self-hosting/vm/).

If you haven't already, please review the [main LiveKit deployment guide](README.md) for
prerequisites, firewall rules, and DNS configuration before proceeding. It covers the groundwork
needed before you start generating configs here.

## Configuration Generation

LiveKit provides a nifty Docker command that generates the basic configuration files you need, tailored
to your domain. Run this command on your local machine or the server where you'll host LiveKit:

```bash
# Pull the generator image
docker pull livekit/generate

# Run the generator, replacing 'livekit.your.domain' with your actual domain
# This will create a folder named 'livekit.your.domain' in your current directory
docker run --rm -it -v $PWD:/output livekit/generate livekit.your.domain
```

This spits out a directory named after the domain you gave it (e.g. `livekit.your.domain/`)
containing these essential files:

- `caddy.yaml`
- `docker-compose.yaml`
- `livekit.yaml`
- `redis.conf`
- `init_script.sh` OR `cloud_init.xxxx.yaml`

## Deployment using Docker Compose

1. **Transfer Files:** Get the generated configuration files (`caddy.yaml`, `docker-compose.yaml`,
   `livekit.yaml`, `redis.conf`) onto your target server. A good spot is often `/opt/livekit/`, but
   choose what makes sense for you.
2. **Review `docker-compose.yaml` (Optional but Recommended):** It's worth taking a quick look inside
   the generated `docker-compose.yaml`. You'll see it sets up the LiveKit server, Redis, and Caddy
   containers. Caddy will automatically obtain and manage TLS certificates from Let's Encrypt or
   ZeroSSL (provided your DNS and firewall are correct!) and handle proxying requests to LiveKit.

   You might consider adding or adjusting resource limits (`cpus`, `mem_limit`) based on your
   server's specifications, similar to how you might configure Synapse or Continuwuity. The defaults
   are usually fine to start with, though.
3. **Start Services:** Change into the directory where you put the files (e.g. `cd /opt/livekit/`)
   and fire everything up:

    ```bash
    cd /opt/livekit/
    docker compose up -d
    ```

With the services running, return to the [main LiveKit deployment guide](README.md) to configure
Matrix integration and learn about upgrading and troubleshooting.
