# n8n

Self-hosted [n8n](https://n8n.io) for a single host, as docker containers.

- Behind a traefik host proxy, which terminates TLS. n8n holds no certificate and
  publishes no port.
- Backed by a shared Postgres engine, rather than the default SQLite file.

## Naming

Docker-only, like the rest of this host. Nothing is installed on the host itself; every
part runs as a container driven by docker compose.

The two infrastructure repos this one depends on carry a **`d`** for that reason —
`traefikd_host_proxy` and `postgresd_host_engine` — where the `d` marks a docker-only
deployment and `_host_` marks host-wide infrastructure shared by every stack. This repo is
one application, so it carries neither marker.

## Status

Not built yet. The plan lives in the issues of this repo:

- **[n8n behind the host proxy, on the shared Postgres engine](../../issues/1)** — the
  map. Read it first.
- Its child issues are the open decisions. Take one from the frontier: an open issue with
  nothing blocking it.

The shared Postgres engine must exist first. That is a separate effort, in the
`postgresd_host_engine` repo.

## Keep this repo generic

This repo is **public**. Every concrete value — the domain, the subdomain, the encryption
key, the database credentials — belongs in `.env`, which is gitignored, or in private
notes.

Never commit a real domain, subdomain, host address, email, or credential. Use
`example.com` and a `${VARIABLE}` placeholder. The same rule applies to the issues.

## Usage

Written once the stack exists. See **[Build the n8n repo and start it behind the
proxy](../../issues/5)**.
