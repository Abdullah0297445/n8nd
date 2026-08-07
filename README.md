# n8n

Self-hosted [n8n](https://n8n.io) for a single host, as docker containers.

- Behind a traefik host proxy, which terminates TLS. n8n holds no certificate and
  publishes no port.
- Backed by a shared Postgres engine, rather than the default SQLite file.

## Naming

`n8nd` — the **`d`** marks a **docker**-only deployment. n8n runs as a container, driven
by docker compose. Nothing is installed on the host itself.

There is no `_host_` marker. That marker belongs to host-wide infrastructure shared by
every stack on the box, such as `traefikd_host_proxy` and `postgresd_host_engine`. This
repo is one application, so it carries the `d` alone.

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
