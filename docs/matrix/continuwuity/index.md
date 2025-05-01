# Matrix Continuwuity Homeserver Guides

This section provides comprehensive guides for deploying Continuwuity, a featureful fork of the Conduit
Matrix homeserver. Written in Rust, Continuwuity aims to be a high-performance and efficient homeserver
that's easy to set up and "just works".

## Quick Start

These Docker guides will walk you through:

1. [Docker Deployment](docker.md) - Set up the Continuwuity container
2. [Server Configuration](config.md) - Configure your homeserver
3. [Reverse Proxies](reverse-proxies/index.md) - Set up external access
   - [SSL Certificates](reverse-proxies/ssl.md) - Secure your server
   - Choose your proxy:
     - [Caddy](reverse-proxies/caddy.md) - Simple, automatic HTTPS
     - [Nginx](reverse-proxies/nginx.md) - Popular and flexible

## Deployment Options

While these guides focus on Docker deployment, Continuwuity provides several installation options:

- **Docker containers** (covered in this guide)
- **Debian packages** (.deb) for x86_64 and ARM64
- **Static binaries** for Linux (x86_64/ARM64) and macOS (x86_64/ARM64)

You can find all these options in the [official releases](https://forgejo.ellis.link/continuwuation/continuwuity/releases).
For non-Docker deployments, refer to the [deployment documentation](https://continuwuity.org/deploying)
which covers setting up users, systemd services, and more.

Continuwuity is quite stable and very usable as a daily driver for low-medium sized homeservers.
While technically in Beta (inherited from Conduit), this status is becoming less relevant as the
codebase significantly diverges from upstream Conduit.

Key features and differences from Conduit:

- Written in Rust for high performance and memory efficiency
- Complete drop-in replacement for Conduit (when using RocksDB)
- Single-process architecture (no worker configuration needed)
- Actively maintained with regular updates
- Designed for stability and real-world use

## Getting Help

If you need assistance, you can join the official Matrix rooms:

- [#continuwuity:continuwuity.org](https://matrix.to/#/#continuwuity:continuwuity.org) -
  Main support and discussion
- [#offtopic:continuwuity.org](https://matrix.to/#/#offtopic:continuwuity.org) -
  Off-topic community chat
- [#dev:continuwuity.org](https://matrix.to/#/#dev:continuwuity.org) -
  Development discussion

Please review the [Continuwuity Community Guidelines](https://continuwuity.org/community)
before participating.

## Try It Out

While there isn't an official public server with open registration currently, you can:

- Try their official mirror of the [Element Web client](https://element.continuwuity.org/)
  (other clients may be available later).
- Check [servers.joinmatrix.org](https://servers.joinmatrix.org) for public homeservers potentially
  running Continuwuity.
- Deploy your own instance by following these guides!

Let's get started with deploying your own efficient Matrix homeserver!
