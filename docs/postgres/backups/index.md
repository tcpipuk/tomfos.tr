---
title: Introduction to PostgreSQL Backups
description: Backing up isn't just good sense, it's essential. Discover why and how to safeguard your PostgreSQL data, especially for crucial applications like Matrix Synapse.
---

# Setting Up a Replica for Backups for PostgreSQL in Docker

Backing up a write-heavy database like Synapse can be a challenge: in my case, a dump of the
database would take >15 minutes and cause all sorts of performance and locking issues in the
process.

This guide will walk you through setting up replication from a primary PostgreSQL Docker container
to a secondary container dedicated for backups.

By the end, you'll have a backup system that's efficient, minimizes performance hits, and ensures
data safety.
