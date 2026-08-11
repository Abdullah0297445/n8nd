# n8n

Self-hosted [n8n](https://n8n.io) for a single host, as docker containers.

- Behind a traefik host proxy, which terminates TLS. n8n holds no certificate and
  publishes no port.
- Backed by a shared Postgres engine, rather than the default SQLite file.

## Naming

`n8nd` — the **`d`** marks a **docker**-only deployment. n8n runs as a container, driven
by docker compose. Nothing is installed on the host itself.

There is no `_host_` marker. That marker belongs to host-wide infrastructure shared by
every stack on a host, such as a reverse proxy or a shared database server. This repo is
one application, so it carries the `d` alone.

## What runs here

| Container | Image | Job |
|---|---|---|
| `n8n` | `docker.n8n.io/n8nio/n8n` | The editor, the webhooks, the schedules. One main process. |
| `n8n-runners` | `docker.io/n8nio/runners` | Every Code node, in its own container. |

The two images come from **two registries**, and that is not a mistake. `docker.n8n.io`
mirrors `n8nio/n8n` alone — it answers `NAME_UNKNOWN` for `n8nio/runners`. The runners image
comes from Docker Hub.

Neither publishes a host port. Only `n8n` joins the proxy network and the Postgres network.
`n8n-runners` reaches the task broker and nothing else.

## Before you start

Three things must already exist. None of them is made by this repo.

1. **The host proxy runs**, so the `traefik_host_network` network exists.
2. **A Postgres server answers on the `postgres_host_network` network**, and it holds a
   database and a login role for n8n. Keep the connection details.
3. **A public DNS A record** for the subdomain points at this host. The proxy uses a
   DNS-01 challenge, so it can issue the certificate before that record exists — but
   nothing answers on the name until it does.

Then run one statement on that server, as a superuser, against the n8n database:

```sql
ALTER ROLE <the n8n role> SET statement_timeout = '5min';
```

n8n runs with `DB_POSTGRESDB_STATEMENT_TIMEOUT=0`, because a transaction pooler drops the
setting anyway. That removes n8n's own cap. Without the statement above, one runaway query
holds a server connection with no limit — on a server that other applications may share.

## Usage

1. `cp .env.example .env`
2. Change every value in `.env`. Generate the two secrets with `openssl rand -hex 32`.
3. Confirm `.gitignore` excludes `.env` **before** the first commit.
4. **Back up `.env` to a secure place away from this host, and back it up again after
   every change to it.** It holds `N8N_ENCRYPTION_KEY`, which is not in the database and
   cannot be derived. Lose that key and every saved credential stops decrypting, for good.
5. `docker compose up -d`
6. `docker compose ps` — both containers must reach a healthy or running state, and
   neither may show a host port binding.
7. `docker logs -f traefik`, on the proxy, to watch the certificate come in. Issuance is
   visible if the proxy logs at `INFO`.
8. Open the subdomain over HTTPS and make the owner account.

## Upgrading

An upgrade runs the migrations of the new version at start. A failure is fatal, and about
one migration in three has no `down()`. So an upgrade is an irreversible schema change,
and it never runs by itself: the tag is exact, never `latest` or `stable`.

Every version bump, in this order:

1. Read every breaking-changes entry between the two versions.
2. Take a **fresh** `pg_dump` of the n8n database. Whatever schedule backs that database
   up, "last night" is not "one minute ago".
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

**This stack backs nothing up itself.** That is a decision, not an omission.

**The database** holds the workflows, the credentials and the execution data. It lives on
a Postgres server that this stack does not own, so backing it up belongs to whoever runs
that server.

**`N8N_ENCRYPTION_KEY` is yours to keep**, and it is in `.env`. A database backup taken
without it restores credentials that nothing can read. Back up `.env` — see step 4 of
[Usage](#usage).

**The `n8n_data` volume needs no backup.** Everything in it rebuilds itself:

| In the volume | Comes back from |
|---|---|
| `.n8n/storage` — binary data of an execution | nothing, and it needs nothing: n8n prunes it together with the execution that owns it, so none of it outlives `EXECUTIONS_DATA_MAX_AGE` |
| the settings file | `.env`, because `N8N_ENCRYPTION_KEY` is pinned there |
| the node cache | itself, at start |
| community nodes | n8n's own database record, because `N8N_REINSTALL_MISSING_PACKAGES` is `true` |

**A file a workflow must keep is the workflow's job.** Write it to durable storage — an
object store, a database — from the workflow itself. Never leave it in `.n8n/storage` and
expect to find it later: n8n deletes from there on its own schedule.

## Notes

- No real subdomain, host address, email or secret belongs in a tracked file. Use
  `example.com` and a `${VARIABLE}`, and keep every real value in `.env`.
- `.env` is gitignored. Never commit it.
- `DB_POSTGRESDB_HOST` names a **network alias**, not a container. Where a connection
  pooler stands in front of Postgres, that alias must reach a **transaction**-pooled
  endpoint — which is what `DB_POSTGRESDB_STATEMENT_TIMEOUT=0` exists for.
- The **Postgres Trigger** node holds its own credential and it uses `LISTEN`, which a
  transaction pooler drops. Point that credential at a **session**-pooled endpoint, or at
  Postgres directly — never at `DB_POSTGRESDB_HOST`.
