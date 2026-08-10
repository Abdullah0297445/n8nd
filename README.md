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

## What runs here

| Container | Image | Job |
|---|---|---|
| `n8n` | `docker.n8n.io/n8nio/n8n` | The editor, the webhooks, the schedules. One main process. |
| `n8n-runners` | `docker.io/n8nio/runners` | Every Code node, in its own container. |

The two images come from **two registries**, and that is not a mistake. `docker.n8n.io`
mirrors `n8nio/n8n` alone — it answers `NAME_UNKNOWN` for `n8nio/runners`. The runners image
comes from Docker Hub.

Neither publishes a host port. Only `n8n` joins the proxy network and the engine network.
`n8n-runners` reaches the task broker and nothing else.

## Before you start

Three things must already exist. None of them is made by this repo.

1. **The host proxy runs**, so the `traefik_host_network` network exists.
2. **The shared Postgres engine runs**, and this stack has a tenant on it:
   `make add-tenant NAME=<name>` in `postgresd_host_engine`. Keep the printed DSN.
3. **A public DNS A record** for the subdomain points at this host. The proxy uses a
   DNS-01 challenge, so it can issue the certificate before that record exists — but
   nothing answers on the name until it does.

Then run one statement on the engine, as the engine superuser, against the tenant
database:

```sql
ALTER ROLE <the n8n role> SET statement_timeout = '5min';
```

n8n runs with `DB_POSTGRESDB_STATEMENT_TIMEOUT=0`, because the transaction door drops the
setting anyway. That removes n8n's own cap. Without the statement above, one runaway query
holds a server connection with no limit, on an engine that other tenants share.

## Usage

1. `cp .env.example .env`
2. Change every value in `.env`. Generate the two secrets with `openssl rand -hex 32`.
   Put a second copy of `N8N_ENCRYPTION_KEY` in a password manager, apart from the
   database archive. Lose that key and every saved credential stops decrypting, for good.
3. Confirm `.gitignore` excludes `.env` **before** the first commit.
4. `docker compose up -d`
5. `docker compose ps` — both containers must reach a healthy or running state, and
   neither may show a host port binding.
6. `docker logs -f traefik`, in the proxy repo, to watch the certificate come in. The
   proxy logs at `INFO`, so ACME issuance is visible.
7. Open the subdomain over HTTPS and make the owner account.

## Upgrading

An upgrade runs the migrations of the new version at start. A failure is fatal, and about
one migration in three has no `down()`. So an upgrade is an irreversible schema change,
and it never runs by itself: the tag is exact, never `latest` or `stable`.

Every version bump, in this order:

1. Read every breaking-changes entry between the two versions.
2. Take a **fresh** `pg_dump` of the n8n database. The engine writes a nightly archive of
   every tenant database, so one exists — but "last night" is not "one minute ago".
3. Edit **both** image tags, `n8n` and `n8n-runners`, to the same version. A runner on a
   different version than n8n is not supported.
4. `docker compose up -d`, and watch the migration in the logs.

A downgrade is a restore from the dump. Treat `db:revert` as unavailable: it undoes one
migration, only from the new image, and it refuses on an irreversible one.

## Health

`n8n` has a healthcheck on `/healthz/readiness`. That endpoint answers 200 only when the
database is connected, the migrations are done, and the start has finished. `/healthz`
answers ok at all times and says nothing about the database, so it is the wrong one to
watch. `n8n-runners` waits for that healthcheck before it starts.

## What is backed up, and what is not

The engine writes a nightly archive of the tenant database. That archive holds the
workflows, the credentials and the execution data.

It does **not** hold the `n8n_data` volume, which is the binary data of an execution, and
it does not hold `N8N_ENCRYPTION_KEY`. An archive without that key restores credentials
that nothing can read. How both are covered is
**[Decide how the n8n volume and the encryption key are backed up](../../issues/6)**.

## Notes

- This repo is **public**. No real subdomain, host address, email or secret may enter any
  file here. Use `example.com` and a `${VARIABLE}`. Grep the tree before every push.
- `.env` is gitignored. Never commit it.
- The DSN names a **door** on the engine, by network alias. `engine` is the transaction
  door, and `engine-session` is the session door. `DB_POSTGRESDB_HOST` in
  `docker-compose.yml` is the one word that moves n8n between them.
- The **Postgres Trigger** node holds its own credential and it uses `LISTEN`, which the
  transaction door drops. Point that credential at the session door, and not at
  `DB_POSTGRESDB_HOST`.
