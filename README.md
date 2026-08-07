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

## Usage

Written once the stack exists. See **[Build the n8nd repo and start it behind the
proxy](../../issues/5)**.
