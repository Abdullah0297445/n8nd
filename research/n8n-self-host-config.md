# Self-hosted n8n behind a TLS-terminating proxy, on a shared Postgres engine

**Applies to:** n8n **2.33.7** (the `stable` channel as of 2026-08-09, published 2026-08-07).
Source facts below were read at git tag `n8n@2.33.7` (commit `e21f9f1`) unless noted.
n8n is on the **2.x** line; the 1.x line is still patched separately (`1.123.69`, 2026-08-05).
Release list: <https://github.com/n8n-io/n8n/releases>

Two conventions used throughout:

- **"docs say"** = stated on docs.n8n.io. **"source shows"** = read from the n8n source at the tag above.
  Where the two disagree, the source wins and the disagreement is called out.
- `${...}` are placeholders for deployment-specific values. `n8n.example.com` stands in for the real public hostname.

**Question → section.** The decision in §2 is pulled to the front because everything else depends on it, so section numbers run one ahead of the question numbers from §3 on:

| Question | Section |
|---|---|
| 1. `DB_POSTGRESDB_*` set, schema creation | §3.1, §3.2 |
| 2. Pooler: transaction or session door | **§2** (verdict), §3.3–§3.6 (evidence) |
| 3. Public URL variables | §4 |
| 4. Trusting the proxy / `N8N_PROXY_HOPS` | §5 |
| 5. `N8N_ENCRYPTION_KEY` | §6 |
| 6. The `/home/node/.n8n` volume, binary data | §7 |
| 7. Task runners | §8 |
| 8. Execution pruning | §9 |
| 9. `GENERIC_TIMEZONE` against `TZ` | §10 |
| 10. Version pinning, upgrade, downgrade | §11 |
| 11. Container port and the traefik label | §12 |
| 12. `linux/amd64` and the image reference | §13 |
| One-way doors | §14 |

---

## 1. Recommended environment

### 1.1 On the `n8n` container

| Variable | Value | Reason |
|---|---|---|
| `DB_TYPE` | `postgresdb` | Selects the Postgres backend. Default is `sqlite`. ([docs][db-env], [source][db-config]) |
| `DB_POSTGRESDB_HOST` | `${PGBOUNCER_HOST}` | The pgbouncer service on the shared docker network. Default `localhost`. ([docs][db-env]) |
| `DB_POSTGRESDB_PORT` | `${PGBOUNCER_PORT}` | Default `5432`. ([docs][db-env]) |
| `DB_POSTGRESDB_DATABASE` | `${N8N_DB_NAME}` | n8n's own database in the shared engine. Default `n8n`. ([docs][db-env]) |
| `DB_POSTGRESDB_USER` | `${N8N_DB_USER}` | n8n's own login role. Default `postgres`. ([docs][db-env]) |
| `DB_POSTGRESDB_PASSWORD` | `${N8N_DB_PASSWORD}` | No default. Pass via docker secret or `.env`, never inline in compose. ([docs][db-env]) |
| `DB_POSTGRESDB_SCHEMA` | `public` | Keep the default. Under transaction pooling the driver's `SET search_path` does not persist — see §3.3. ([docs][db-env]) |
| `DB_POSTGRESDB_POOL_SIZE` | `2` (default; leave unset) | Client-side pg-pool `max`, **per n8n process**. Every connection consumes a pgbouncer client slot. ([docs][db-env], [source][db-config]) |
| `DB_POSTGRESDB_STATEMENT_TIMEOUT` | `0` | **Required for the transaction door.** Non-zero makes the driver issue a bare `SET statement_timeout` on every pool acquisition, which transaction pooling silently drops and leaks. Enforce the cap with `ALTER ROLE` instead. Undocumented; default is `300000` (5 min). ([source][db-config], [source][pg-driver]) |
| `N8N_HOST` | `n8n.example.com` | Feeds the fallback base URL. Belt-and-braces once the two URL variables below are set. Default `localhost`. ([docs][deploy-env], [source][url-svc]) |
| `N8N_PROTOCOL` | `https` | Same. Default `http`. ([docs][deploy-env]) |
| `N8N_PORT` | *unset* (`5678`) | The in-container listen port. Leave it alone; traefik routes to `5678`. See §11. ([docs][deploy-env], [source][dockerfile]) |
| `N8N_WEBHOOK_URL` | `https://n8n.example.com/` | Public base for production **and** test webhook URLs. Successor to `WEBHOOK_URL`. ([docs][webhook-proxy], [source][url-svc]) |
| `N8N_EDITOR_BASE_URL` | `https://n8n.example.com/` | Public base for the editor, OAuth callback URLs and outbound emails. ([docs][deploy-env], [source][url-svc]) |
| `N8N_PROXY_HOPS` | `1` | Exactly one proxy in front (traefik). Makes express `trust proxy` honour `X-Forwarded-*`. Default `0`. ([docs][webhook-proxy], [source][proxy-hops]) |
| `N8N_SECURE_COOKIE` | *unset* (`true`) | Default is already `true`; correct when the browser reaches n8n over HTTPS. ([source][auth-config]) |
| `N8N_ENCRYPTION_KEY` | `${N8N_ENCRYPTION_KEY}` | **Pin it.** Unset means n8n generates one into the volume; losing the volume loses every credential. See §5. ([docs][enc-key], [source][instance-settings]) |
| `N8N_DEFAULT_BINARY_DATA_MODE` | `filesystem` | Already the 2.x default in regular mode, but pin it so a future default change cannot strand data. See §6. ([source][binary-config], [docs][v2-breaking]) |
| `N8N_RUNNERS_MODE` | `external` | Runs the Code node in a separate container. `internal` (the default) is "not recommended for production" per docs. See §7. ([docs][runners-setup], [source][runners-config]) |
| `N8N_RUNNERS_AUTH_TOKEN` | `${N8N_RUNNERS_AUTH_TOKEN}` | Shared secret between broker and runner. Required in `external` mode. ([docs][runners-env]) |
| `N8N_RUNNERS_BROKER_LISTEN_ADDRESS` | `0.0.0.0` | Default `127.0.0.1` is unreachable from the sibling runner container. ([docs][runners-env]) |
| `EXECUTIONS_DATA_PRUNE` | `true` (default; set explicitly) | Rolling deletion of old executions. Critical in a shared engine. See §8. ([source][exec-config]) |
| `EXECUTIONS_DATA_MAX_AGE` | `336` (14 days) or lower | Age in hours before an execution is soft-deleted. ([source][exec-config]) |
| `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `10000` or lower | Row-count ceiling. `0` means unlimited — never set that here. ([source][exec-config]) |
| `GENERIC_TIMEZONE` | `${TZ}` | The **workflow** timezone: what Schedule/Cron nodes mean by "03:00". Default `America/New_York`. See §9. ([docs][tz-env]) |
| `TZ` | `${TZ}` | The **container/OS** timezone: log timestamps and `Date` in the Code node. See §9. ([docs][docker-install]) |

### 1.2 On the `n8n-runners` sidecar

| Variable | Value | Reason |
|---|---|---|
| `N8N_RUNNERS_TASK_BROKER_URI` | `http://n8n:5679` | The main container's broker port. Default `http://127.0.0.1:5679`. ([docs][runners-env]) |
| `N8N_RUNNERS_AUTH_TOKEN` | `${N8N_RUNNERS_AUTH_TOKEN}` | Must match the main container exactly. ([docs][runners-env]) |

Image: `docker.n8n.io/n8nio/runners:2.33.7` — the runner image tag **must match** the n8n image tag ([docs][runners-setup]).

### 1.3 On the shared Postgres engine (not n8n env, but part of the answer)

| Setting | Value | Reason |
|---|---|---|
| pgbouncer `pool_mode` | `transaction` | See §2. |
| `ALTER ROLE ${N8N_DB_USER} SET statement_timeout = '5min'` | — | Replaces the driver-side `SET` that transaction pooling drops. A role default is applied at server-session start, so it survives pooling. |
| pgbouncer `default_pool_size` | `≥ (n8n processes × DB_POSTGRESDB_POOL_SIZE) + 2` | Migrations pin one server connection for their whole run; leave headroom. |
| Postgres major version | 16, 17 or 18 | Docs: n8n supports the latest two actively maintained majors (17, 18) plus 16. Derivatives (Aurora, AlloyDB, CockroachDB, YugabyteDB) are **not** supported. ([docs][choose-db]) |

### 1.4 Traefik label

```
traefik.http.services.n8n.loadbalancer.server.port=5678
```

---

## 2. THE DECISION: transaction door or session door?

> ### Recommendation: **the transaction door.**
> Confidence: **high**, conditional on setting `DB_POSTGRESDB_STATEMENT_TIMEOUT=0`.

n8n 2.x is written to be pooler-safe on purpose. The evidence is in §3; the residual risk is in §3.6.

**One exception, and it is not about n8n's own database:** if any workflow uses the **Postgres Trigger** node, that node opens its *own* connection using *its own* credential and does use `LISTEN`/`pg_notify` ([source][pg-trigger]). That credential must point at the **session door** (or straight at the engine). This is a per-credential choice inside n8n, entirely separate from `DB_POSTGRESDB_HOST`.

---

## 3. `DB_POSTGRESDB_*`, migrations, and pooler compatibility

### 3.1 The full variable set

All of the following are read by `DatabaseConfig` ([source][db-config]) and documented at [docs][db-env], except `DB_POSTGRESDB_STATEMENT_TIMEOUT` and `DB_POSTGRESDB_DESTROY_TIMEOUT_MS`, which exist in source but are **absent from the docs page**.

| Variable | Default | Notes |
|---|---|---|
| `DB_TYPE` | `sqlite` | `sqlite` \| `postgresdb` |
| `DB_TABLE_PREFIX` | `''` | Prefixes every table name. Source comment: "useful for shared databases". |
| `DB_POSTGRESDB_DATABASE` | `n8n` | |
| `DB_POSTGRESDB_HOST` | `localhost` | |
| `DB_POSTGRESDB_PORT` | `5432` | |
| `DB_POSTGRESDB_USER` | `postgres` | |
| `DB_POSTGRESDB_PASSWORD` | `''` | |
| `DB_POSTGRESDB_SCHEMA` | `public` | |
| `DB_POSTGRESDB_POOL_SIZE` | `2` | → pg-pool `max`, per process |
| `DB_POSTGRESDB_CONNECTION_TIMEOUT` | `20000` ms | Also reused as pg-pool `connectionTimeoutMillis` |
| `DB_POSTGRESDB_IDLE_CONNECTION_TIMEOUT` | `30000` ms | Client-side idle eviction |
| `DB_POSTGRESDB_STATEMENT_TIMEOUT` | `300000` ms | **Undocumented.** Issued as a bare `SET` — see §3.3 |
| `DB_POSTGRESDB_MAX_CONNECTION_LIFETIME_MS` | `3600000` ms | → pg-pool `maxLifetimeSeconds`; `0` disables |
| `DB_POSTGRESDB_DESTROY_TIMEOUT_MS` | `10000` ms | **Undocumented.** Force-close window during pool recovery |
| `DB_POSTGRESDB_KEEP_ALIVE` | `true` | TCP keep-alive |
| `DB_POSTGRESDB_KEEP_ALIVE_INITIAL_DELAY_MS` | `10000` ms | |
| `DB_POSTGRESDB_SSL_ENABLED` | `false` | |
| `DB_POSTGRESDB_SSL_CA` / `_CERT` / `_KEY` | `''` | Setting any of these turns SSL on implicitly |
| `DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED` | `true` | |

Engine-agnostic connection-health variables, new in the 2.x line ([docs][db-env], [source][db-config]):
`DB_PING_INTERVAL_SECONDS` (`2`), `DB_PING_TIMEOUT_MS` (`5000`), `DB_PING_MAX_FAILURES_BEFORE_RECOVERY` (`3`), `DB_RECOVERY_BACKOFF_MIN_MS` (`1000`), `DB_RECOVERY_BACKOFF_MAX_MS` (`30000`), `DB_CONNECTION_ACQUISITION_TIMEOUT_MS` (`30000`).
The legacy `N8N_DB_PING_TIMEOUT` is deprecated and logs a warning ([source][db-connection]).

### 3.2 Schema creation — automatic, no hand-run migrations

n8n creates and migrates its own schema on every start. **Inferred from source**, and consistent with the docs statement that "n8n needs to create and modify the schemas of the tables it uses" ([docs][choose-db]):

- The DataSource is built with `migrationsRun: false` and `synchronize: false` ([source][db-conn-options]) — so TypeORM's own auto-migrate is off.
- Instead `BaseCommand` calls `dbConnection.migrate()` explicitly during startup, immediately after `init()`, and a failure is fatal: `exitWithCrash('There was an error running database migrations', error)` ([source][base-command]).

So: **nothing to run by hand.** The role needs DDL rights on its schema. Consequence for upgrades: an image swap silently migrates the database on next boot — see §10 and the one-way-door list.

### 3.3 Prepared statements — not a problem

- node-postgres prepares server-side **only when a query carries a `name`**. Its docs: "If you supply a `name` parameter the query execution plan will be cached on the PostgreSQL server on a per connection basis." ([node-postgres][npg-queries])
- n8n vendors its own TypeORM fork at `packages/@n8n/typeorm`. `PostgresQueryRunner.query()` calls `databaseConnection.query(query, parameters)` — text and parameters only, never a `name` ([source][pg-runner]). So every query uses the **unnamed** extended-protocol statement, which is Sync-bounded and legal under transaction pooling.
- Belt and braces: pgbouncer has supported protocol-level named prepared statements in transaction mode since `max_prepared_statements` was introduced, and its default is `200` ([pgbouncer][pgb-config]).

### 3.4 Session-scoped state — deliberately avoided

| Mechanism | Does n8n use it on its own DB? | Evidence |
|---|---|---|
| Session advisory locks (`pg_advisory_lock`) | **No** | `DbLockService` uses only `pg_advisory_xact_lock` / `pg_try_advisory_xact_lock` ([source][db-lock]) |
| Transaction advisory locks | Yes, extensively | Same file. Lock IDs are centrally registered: `AUTH_ROLES_SYNC`, `TRUSTED_KEY_REFRESH`, `WORKFLOW_STATISTICS_ROLLUP`, `WORKFLOW_REVIEW_REQUEST_CREATE`, `MIGRATIONS`, `EVAL_COLLECTION_RERUN`, `INSTANCE_AI_SETTINGS` |
| `LISTEN` / `NOTIFY` | **No** | Repo-wide, `pg_notify` appears only in the Postgres **Trigger node** ([source][pg-trigger]) |
| Pub/sub, leader election (multi-main) | **Redis, not Postgres** | `RedisLockService` is wired in only for queue mode / multi-main / redis cache ([source][base-command]) |
| `SET` on the session | Yes — see §3.5 | ([source][pg-driver]) |
| `SET LOCAL` (transaction-scoped) | Yes, and it is correct | ([source][db-lock]) |
| Temp tables | None found | |
| `CREATE INDEX CONCURRENTLY` | **None** in the 57 Postgres migration files | (would fail anyway — migrations run inside one transaction) |
| Health ping | `SELECT 1` — pooler-safe | ([source][db-monitor]) |

The design intent is explicit. `DbLockService`'s own doc comment reads:

> "Transaction-scoped locks are the safer choice in connection-pooled environments (like TypeORM): a session-scoped lock that is not explicitly released before the connection returns to the pool would remain held when the pool hands that connection to the next caller, causing unexpected blocking or deadlocks." ([source][db-lock])

Migrations run under `pg_advisory_xact_lock` with `waitIndefinitely: true`, which first issues `SET LOCAL statement_timeout = 0`, `SET LOCAL lock_timeout = 0` and `SET LOCAL idle_in_transaction_session_timeout = 0` — all transaction-scoped, all correct under a pooler ([source][db-lock], [source][db-connection]).

### 3.5 The one real defect

`PostgresDriver.obtainMasterConnection()` runs on **every** pool acquisition, *before* any `BEGIN` ([source][pg-driver]):

```
SET search_path TO public          -- or  SET search_path TO "<schema>",public
SET statement_timeout = 300000     -- only when DB_POSTGRESDB_STATEMENT_TIMEOUT != 0
```

pgbouncer's own feature matrix marks `SET`/`RESET` as **"Never"** supported under transaction pooling, and `server_reset_query` is not run in transaction mode (`server_reset_query_always` defaults to `0`) ([pgbouncer][pgb-features], [pgbouncer][pgb-config]).

Consequences under the transaction door:

1. `DB_POSTGRESDB_STATEMENT_TIMEOUT` **silently does nothing** — the safety cap you think you have, you do not have.
2. Both settings **leak** onto whichever server connection ran them, for the next borrower. Contained blast radius: pgbouncer keys pools by `(database, user)`, and n8n has its own database and its own role, so the leak stays inside n8n's own pool and cannot reach another tenant.
3. Two extra server round-trips per logical database operation.
4. `search_path` is a non-issue at `DB_POSTGRESDB_SCHEMA=public`: the statement sets it to the value it already has. It would matter for a non-public schema — mitigated because TypeORM schema-qualifies table names, but see §3.6.

**Mitigation, and why it is clean:** set `DB_POSTGRESDB_STATEMENT_TIMEOUT=0`. The driver then skips the `SET` entirely, and `ALTER ROLE ${N8N_DB_USER} SET statement_timeout = '5min'` gives the same protection through a mechanism that *is* pooling-safe (role defaults apply when the server session starts). Migrations are unaffected: with the env var at `0` the migration path leaves `statement_timeout = 0` from its `SET LOCAL`, so long migrations are not cut off, and `SET LOCAL` overrides the role default for that transaction.

This is not a startup-packet problem. `createPool()` passes only `connectionString, host, user, password, database, port, ssl, connectionTimeoutMillis, application_name, max, query_timeout` plus the `extra` bag to `pg`, so `statement_timeout` never reaches the startup packet. `KEEP_ALIVE`, `maxConnectionLifetimeMs` and `idleTimeoutMs` all ride in `extra` as client-side pg-pool knobs with zero wire impact ([source][db-conn-options], [source][pg-driver]).

### 3.6 What the docs say, what the maintainers say, and residual risk

**Docs:** pgbouncer, "connection pooler", "transaction pooling" and the Supabase pooler appear **nowhere** in the n8n documentation corpus. `DB_POSTGRESDB_POOL_SIZE` is documented (Number, default `2`, "Control how many parallel open Postgres connections n8n should have") but never discussed in relation to an external pooler ([docs][db-env]).

**Maintainers:** issue [#26735 "pgBouncer Transaction Pooling Incompatibility"][issue-26735] (opened 2026-03-08, closed 2026-03-30) is the only pooler report in the tracker. The reporter blamed advisory locks. A maintainer rebutted that "n8n uses `pg_advisory_xact_lock` … the used advisory locking mechanism in n8n does not require session-level persistence" and could not reproduce with `pool_mode=transaction` across multiple mains and workers. The reporter then retracted: the real cause was "startup parameter rejection … and idle-in-transaction timeouts", fixed with `DB_POSTGRESDB_STATEMENT_TIMEOUT=0` and `PGOPTIONS`, concluding "We're running Odyssey in transaction pooling mode and it's been stable since." That closure is the direct precedent for the mitigation in §3.5.

Issue [#30612][issue-30612] / PR [#31008][pr-31008] (May 2026) concerned pool congestion after a database-proxy outage and produced the ping/recovery machinery in §3.1 — a resilience gap, not a pooling incompatibility.

**Residual risk (accept these, or move to the session door):**

- Whether any of the 57 raw-SQL Postgres migrations relies on `search_path` rather than a qualified name — **not established** without a per-file audit. Irrelevant while `DB_POSTGRESDB_SCHEMA=public`.
- Whether n8n's CI tests against any pooler — **not established**; no workflow or doc evidence. Transaction-mode support is therefore *by construction and by maintainer statement*, not by a published test matrix.
- A future n8n release could reintroduce session-scoped state; nothing in the docs promises pooler compatibility, so nothing forbids it. Re-check §3.4 on major upgrades.
- The long migration transaction pins one pgbouncer server connection for its full duration. Size `default_pool_size` accordingly, and keep pgbouncer's `query_wait_timeout` in mind for concurrent tenants.

---

## 4. Public URL variables — which ones a TLS-terminating proxy makes necessary

n8n's URL resolution is a three-level fallback chain ([source][url-svc]):

```
baseUrl            = N8N_PROTOCOL://N8N_HOST:N8N_PORT + N8N_PATH
                     (the :port is omitted for http:80 and https:443)

webhookBaseUrl     = N8N_WEBHOOK_URL  ||  WEBHOOK_URL (deprecated)  ||  baseUrl

instanceBaseUrl    = N8N_EDITOR_BASE_URL  ||  webhookBaseUrl
```

n8n publishes no host port and holds no certificate, so `baseUrl` computes to `http://localhost:5678/` — a URL that is wrong for every external consumer. What breaks if each is left unset:

| Variable | Necessary? | What goes wrong when unset |
|---|---|---|
| `N8N_WEBHOOK_URL` | **Yes** | `webhookBaseUrl` falls back to `baseUrl`. The webhook URLs shown in the editor read `http://localhost:5678/webhook/...`, and that is the string users copy into third-party services — so callbacks are configured against an unreachable address. Registration-based triggers that push a callback URL to a provider register the localhost URL too. |
| `N8N_EDITOR_BASE_URL` | **Yes** in principle; **redundant** here | It falls back to `webhookBaseUrl`, so setting `N8N_WEBHOOK_URL` alone already fixes it. Set it explicitly anyway: it is the base for **OAuth callback URLs** and for links in **outbound email** (password reset, invitations), and relying on the fallback couples two unrelated settings. Docs describe it as "Public URL where users can access the editor. Also used for emails sent from n8n." ([docs][deploy-env]) |
| `N8N_PROTOCOL` | Recommended | Only reaches URLs through `baseUrl`, which the two variables above shadow. Left at `http` it makes the fallback wrong, so a future misconfiguration degrades to a broken-but-plausible `http://` URL instead of failing loudly. |
| `N8N_HOST` | Recommended | Same reasoning; default `localhost`. |
| `N8N_PORT` | **No — leave unset** | Changing it changes the port n8n actually listens on inside the container (§11). It does not need to match the public port; `N8N_WEBHOOK_URL` already carries the public 443. |
| `WEBHOOK_URL` | **Do not use** | Deprecated alias of `N8N_WEBHOOK_URL`. Still honoured, but n8n logs a deprecation warning at startup. ([docs][endpoints-env], [source][url-svc]) |

Concretely, with both URL variables set to `https://n8n.example.com/`: OAuth redirect URIs, editor-displayed webhook URLs, and email links all render against the public name, and the internal `5678` never appears in anything a user or a third party sees.

The docs page for this exact scenario prescribes `N8N_WEBHOOK_URL` + `N8N_PROXY_HOPS=1` and requires the proxy to forward `X-Forwarded-For`, `X-Forwarded-Host` and `X-Forwarded-Proto` ([docs][webhook-proxy]). Traefik sets all three by default.

---

## 5. How n8n trusts the proxy — `N8N_PROXY_HOPS`

`N8N_PROXY_HOPS` is the variable; there is no separate `trust proxy` setting. The mapping is direct ([source][proxy-hops]):

```ts
@Env('N8N_PROXY_HOPS')
proxy_hops: number = 0;

// packages/cli/src/abstract-server.ts
const proxyHops = this.globalConfig.proxy_hops;
if (proxyHops > 0) this.app.set('trust proxy', proxyHops);
```

**For exactly one proxy hop: `N8N_PROXY_HOPS=1`.**

The numeric form of express's `trust proxy` means "trust the *n*th-from-last entry in `X-Forwarded-For`". Note the guard: at the default `0`, `trust proxy` is never set at all, so express keeps its default of not trusting any forwarded header.

What breaks when the value is wrong:

- **Too low (`0`, the default, behind one proxy).** n8n does not trust `X-Forwarded-*`. `req.ip` becomes the proxy's own container address, so every request appears to come from one client: rate limiting (the MCP, OAuth-server and JWKS limiters in §1 all count per IP) buckets the whole internet together, and audit/log entries record the proxy instead of the caller. `req.protocol` reads `http` rather than the `https` in `X-Forwarded-Proto`, so anything deriving a URL from the live request — and the `Secure` cookie decision — sees a plain-HTTP request. This is the *safe* direction: headers are ignored, not believed.
- **Too high (e.g. `2` behind one proxy).** n8n walks further left in `X-Forwarded-For` than there are trusted hops, into the portion a **client** controls. A caller sending `X-Forwarded-For: 1.2.3.4` gets `req.ip == 1.2.3.4`. That is a spoofable client identity: rate limits are bypassed by rotating the header, and any IP-based allow/deny logic or audit trail is forgeable. This is the *dangerous* direction — an over-count is a security bug, not a cosmetic one.

Count hops as the number of proxies that **append** to `X-Forwarded-For` between the client and n8n. Here that is traefik alone: **1**. If a CDN or another L7 proxy is ever put in front of traefik, this becomes `2`.

---

## 6. `N8N_ENCRYPTION_KEY`

**What it encrypts.** In 2.x it is the master key of a two-layer scheme: the instance encryption key "protect[s] the data encryption keys", and the data encryption key "directly encrypts your credential data" — credentials, OAuth tokens, and other sensitive content ([docs][rotate-keys]). The older docs phrasing is that it encrypts "the credentials before they get saved to the database" ([docs][enc-key]). It also derives two other secrets, **inferred from source**: the binary-data URL signing secret (`sha256("url-signing:" + encryptionKey)`, unless `N8N_BINARY_DATA_SIGNING_SECRET` is set or a value is already persisted in the database) ([source][binary-config]), and an HMAC signature value and the derived `instanceId` ([source][instance-settings]).

Workflows themselves are **not** encrypted — they are ordinary rows.

**Where it is stored when unset.** n8n generates `randomBytes(24).toString('base64')` on first launch and writes it into the JSON settings file at **`~/.n8n/config`** — in the container, `/home/node/.n8n/config` ([source][instance-settings]; docs say "saves it in the `~/.n8n` folder", [docs][enc-key]). `N8N_ENCRYPTION_KEY_FILE` is also accepted, for the docker-secret pattern ([source][instance-settings-config]).

**When it is not pinned and the volume is lost or the container is recreated without it.** A new random key is generated. Every credential row in Postgres was sealed with the old key and is now undecryptable. Workflows still load; every credential fails at execution time. **Recovery is impossible** without the original key — it is not derivable and not stored in the database.

**When it changes while a settings file already exists.** n8n does not silently proceed. It compares them and throws at startup:

> "Mismatching encryption keys. The encryption key in the settings file `<path>` does not match the N8N_ENCRYPTION_KEY env var. Please make sure both keys match." ([source][instance-settings])

That is the good case — a loud failure. The bad case is a *fresh* volume plus a *different* key: nothing to compare against, so n8n starts cleanly and the damage only surfaces when a credential is used.

**Recovery:** only by restoring the original key value, or by restoring a database backup and re-entering every credential by hand. Note that credentials cannot be exported in plaintext without the key either, so there is no export-then-reimport escape.

**Key rotation** now exists as a supported feature: set `N8N_ENV_FEAT_ENCRYPTION_KEY_ROTATION=true` on all instances, then rotate via **Settings > Data Encryption Keys** or `POST /encryption/keys`. After rotation, new writes use the new key and older records stay readable. But the docs are blunt about the trap: "Removing `N8N_ENV_FEAT_ENCRYPTION_KEY_ROTATION` or setting it to `false` makes all data encrypted after you enabled the feature permanently inaccessible", and "The only recovery path is restoring from a database backup taken before you enabled the feature" ([docs][rotate-keys]). Enabling rotation is itself a one-way door — see §12.

---

## 7. What still lives in `/home/node/.n8n`

**The volume is still needed**, even with Postgres holding workflows, executions and credentials.

Contents, from source ([source][instance-settings], [source][binary-config]):

| Path | Contents | Still used with Postgres? |
|---|---|---|
| `.n8n/config` | JSON settings file: `encryptionKey`, `tunnelSubdomain`, `fsStorageMigrated` | Yes — but pinning `N8N_ENCRYPTION_KEY` makes the key itself reproducible |
| `.n8n/storage/` | **Binary data** in `filesystem` mode, and filesystem execution data | **Yes — this is the load-bearing one** |
| `.n8n/custom/` | Custom nodes and credentials | Yes, if used |
| `.n8n/nodes/` | Installed community nodes | Yes, if used |
| `.n8n/node-definitions/` | Node definition cache | Yes |
| `.n8n/database.sqlite` | The SQLite database | No — Postgres replaces it |

Related: `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS` defaults to **`true`** in 2.x — n8n checks the settings file is not world-readable and tries to `chmod 0600` it ([source][instance-settings-config]). The docs' `docker run` example still passes it explicitly ([docs][docker-install]); that is now redundant but harmless.

**Binary data needs its own setting — and the docs are stale on this point.**

- The env-var reference page still lists `N8N_DEFAULT_BINARY_DATA_MODE` with default `default` and still lists `N8N_AVAILABLE_BINARY_DATA_MODES` ([docs][binary-env]), and the scaling page still says n8n "keeps the data in memory" by default, which "can cause crashes when working with large files" ([docs][binary-scaling]). **Both describe 1.x.**
- The v2.0 breaking-changes page says the in-memory `default` mode was **removed**, leaving `filesystem` (default in regular mode), `database` (default in queue mode) and `s3`; and that `N8N_AVAILABLE_BINARY_DATA_MODES` was **removed** ([docs][v2-breaking]).
- Source at 2.33.7 confirms the breaking-changes page: `this.mode ??= executionsConfig.mode === 'queue' ? 'database' : 'filesystem'`, and `availableModes` is a plain field (no longer an `@Env`) fixed at `['filesystem', 's3', 'database']` ([source][binary-config]).

So on 2.33.7 in regular (non-queue) mode the effective default is already **`filesystem`**, and the old memory-exhaustion consequence no longer applies. Set it explicitly anyway so that switching to queue mode later does not silently move binary data into Postgres — which, in a shared engine, is exactly the growth you are trying to avoid.

The storage path resolution also changed, and the source comment is explicit that the old default is gone ([source][binary-config]):

```
N8N_BINARY_DATA_STORAGE_PATH  →  N8N_STORAGE_PATH  →  ~/.n8n/storage
"`~/.n8n/binaryData` is no longer the default if the env var is unset."
```

`database` mode caps a single file at 512 MiB by default (`N8N_BINARY_DATA_DATABASE_MAX_FILE_SIZE`), hard-limited to 1024 MiB "because of Postgres BYTEA hard limit" ([source][binary-config]).

For this deployment `filesystem` on the volume is right. Move to `s3` only if the volume becomes a constraint — and note that switching modes does **not** migrate existing data, and pruning only touches the currently active mode ([docs][binary-scaling]).

---

## 8. Task runners

**Does current n8n warn or refuse to start without `N8N_RUNNERS_ENABLED`? No — the question no longer applies on 2.x.**

- The docs env-var table lists `N8N_RUNNERS_ENABLED`, Boolean, default `false`, marked "**Deprecated** from v2.0" ([docs][runners-env]).
- The v2.0 breaking-changes page states n8n "enables task runners by default in v2.0" and that "All Code node executions will run on task runners"; `N8N_RUNNERS_ENABLED=true` is offered only as a way to *test the change before upgrading* on 1.x ([docs][v2-breaking]).
- **Confirmed in source:** `TaskRunnersConfig` at 2.33.7 has no `enabled` field at all — the `@Env` declarations begin at `N8N_RUNNERS_MODE` ([source][runners-config]). `BaseCommand` starts the runner module unconditionally for commands that need it (`if (this.needsTaskRunner) { … TaskRunnerModule.start() }`), with no enabled gate ([source][base-command]).

So on 2.33.7 the variable is a **no-op**. Set nothing; on 1.x it was mandatory and n8n warned without it.

**Internal or its own container?** Both work. `N8N_RUNNERS_MODE` defaults to `internal`, which launches the runner as a child process of the n8n container sharing its `uid`/`gid` — the docs call this "not recommended for production". `external` has a launcher manage runners in a separate container (the `n8nio/runners` sidecar), giving "proper isolation between n8n and task runner processes" ([docs][runners-setup]).

**Simplest working external setup** ([docs][runners-setup], [docs][runners-env]):

```
# n8n container
N8N_RUNNERS_MODE=external
N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0     # default 127.0.0.1 is unreachable from a sibling container
N8N_RUNNERS_AUTH_TOKEN=${N8N_RUNNERS_AUTH_TOKEN}

# runners container — image docker.n8n.io/n8nio/runners:2.33.7 (tag MUST match n8n's)
N8N_RUNNERS_TASK_BROKER_URI=http://n8n:5679
N8N_RUNNERS_AUTH_TOKEN=${N8N_RUNNERS_AUTH_TOKEN}
```

The broker port is `5679` (`N8N_RUNNERS_BROKER_PORT`) — distinct from the HTTP port `5678`, and internal to the compose network only. Do not route traefik to it. Useful extras: `N8N_RUNNERS_MAX_CONCURRENCY` (`5`), `N8N_RUNNERS_TASK_TIMEOUT` (`300` s). Note `N8N_RUNNERS_INSECURE_MODE` (`false`) logs a loud production warning if enabled ([source][base-command]).

If nothing sets `N8N_RUNNERS_MODE`, `internal` is used and the Code node still works — external mode is a hardening choice, not a prerequisite.

---

## 9. Execution pruning

**Pruning is on by default in 2.x.** Values below are from source ([source][exec-config]) and match the docs table ([docs][exec-env]):

| Variable | Default | Effect |
|---|---|---|
| `EXECUTIONS_DATA_PRUNE` | `true` | Master switch for rolling deletion |
| `EXECUTIONS_DATA_MAX_AGE` | `336` (hours = 14 days) | Age at which a finished execution qualifies for soft-deletion |
| `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `10000` | Row ceiling. "Does not necessarily prune to the exact max number." `0` = unlimited |
| `EXECUTIONS_DATA_HARD_DELETE_BUFFER` | `1` (hour) | Grace period before a soft-deleted row is physically removed |
| `EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL` | `60` (minutes) | Soft-delete sweep frequency |
| `EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL` | `15` (minutes) | Hard-delete sweep frequency |

**Binary data has no separate pruning variable.** It is pruned as a side effect of execution pruning — but "only prunes binary data in the filesystem" for the *currently active* mode, so data left behind in a previously-used mode is never reclaimed ([docs][binary-scaling]).

Two levers matter more than pruning for a shared engine, because they stop rows being written at all ([docs][exec-env]):

- `EXECUTIONS_DATA_SAVE_ON_SUCCESS` (default `all`) → set to `none` to stop storing successful runs entirely. This is the single biggest reduction available.
- `EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS` (default `true`) and `EXECUTIONS_DATA_SAVE_ON_PROGRESS` (default `false`).

Also new in 2.x: `N8N_EXECUTION_DATA_STORAGE_MODE` (default `database`) can move execution data to `filesystem`, `s3` or `azure`, taking the bulk out of the shared engine while leaving the index rows behind ([docs][exec-env]).

The defaults are sane, but they are *ceilings, not budgets*: 10 000 executions of a workflow with large payloads is still a large table. Lower both `EXECUTIONS_DATA_MAX_AGE` and `EXECUTIONS_DATA_PRUNE_MAX_COUNT` to what you will actually look at, and set them explicitly so a future default change cannot surprise a shared engine.

---

## 10. Timezone: `GENERIC_TIMEZONE` against `TZ`

They control different layers, and the official `docker run` example sets **both** to the same value ([docs][docker-install]):

```shell
-e GENERIC_TIMEZONE="<YOUR_TIMEZONE>" \
-e TZ="<YOUR_TIMEZONE>" \
```

| | Controls | Default |
|---|---|---|
| `GENERIC_TIMEZONE` | The **n8n instance timezone**: "Important for schedule nodes (such as Cron)" — what a workflow means by "run at 03:00", and the default timezone for workflows that do not override it in their own settings | `America/New_York` ([docs][tz-env]) |
| `TZ` | The **container's OS timezone** — the standard glibc/musl variable. Log timestamps, and anything calling `Date`/`toLocaleString` in a Code node or expression | The image sets none, so UTC |

The docs page for `TZ` is thin: the timezone env-var reference documents only `GENERIC_TIMEZONE` and `N8N_DEFAULT_LOCALE` and does not mention `TZ` at all ([docs][tz-env]); the "Set the timezone" example page likewise covers only `GENERIC_TIMEZONE` ([docs][tz-example]). The instruction to set both comes from the Docker installation page ([docs][docker-install]). That `TZ` is the OS-level variable is standard Unix behaviour, not an n8n-specific claim.

What goes wrong if only one is set:

- **Only `TZ`.** Schedule and Cron nodes still fire on `America/New_York`, because `GENERIC_TIMEZONE` keeps its default. A workflow set to "every day at 09:00" runs at 09:00 New York time. This is the dangerous one — it is silent, and the offset drifts twice a year with US daylight saving, independently of the local DST transition.
- **Only `GENERIC_TIMEZONE`.** Schedules are correct, but the container clock stays UTC. Log lines and any `new Date().toString()` in a Code node render in UTC, so timestamps do not line up with the schedules or with the other tenants' logs. Cosmetic and debugging-hostile rather than functionally wrong.

Set both to the same IANA name.

---

## 11. Version pinning, upgrades, and downgrades

**Channels.** The repo carries moving pointer tags and promotes builds into channels via `.github/workflows/release-push-to-channel.yml` ([source][release-workflow]): channel `stable` retags both `n8n:stable` **and** `n8n:latest`; channel `beta` retags both `n8n:beta` **and** `n8n:next`. So `latest` == `stable` and `next` == `beta`. Verified on the registry: `latest` and `stable` resolve to the same digest (2.33.7), `next` and `beta` to another (2.34.4). Docs: "The `stable` version is for production use. `beta` is the most recent release. The `beta` version may be unstable." ([docs][docker-install]). `nightly` / `v3-nightly` tags are built from master — do not use.

**Pin an exact version, not a channel.** `docker.n8n.io/n8nio/n8n:2.33.7`. A channel tag moves under you on roughly weekly promotions, and because n8n migrates its own database at startup (§3.2), an unattended `docker compose pull` is an unattended, irreversible schema change. Digest-suffixed tags (`2.33.7-e21f9f1`) are also published if you want a stronger pin.

**Upgrading across majors.** The docs give process guidance, not a version ladder ([docs][update-n8n]): "Update frequently: this avoids having to jump multiple versions at once, reducing the risk of a disruptive update. Try to update at least once a month"; "Check the Release notes for breaking changes"; test in a separate environment first. Semver policy is "MAJOR version when making incompatible changes which can require user action". Per-major breaking-change pages exist ([docs][v2-breaking]; a v3 page already exists), as does `packages/cli/BREAKING-CHANGES.md` in the repo. **Whether skipping a major is supported is not established** — there is no documented prohibition and no documented requirement to step through each major. In practice you must read every breaking-changes entry between source and target.

Note the release notes moved: docs.n8n.io's release-notes pages now carry "This page is no longer updated… For the latest releases, including every patch version, see the n8n releases on GitHub." GitHub Releases is now the authoritative version list.

**Downgrade after a migration has run: treat it as a ONE-WAY DOOR.**

Partially possible, and only if planned *before* the downgrade:

1. A `db:revert` command exists — `packages/cli/src/commands/db/revert.ts`, "Revert last database migration". It reverts **exactly one** migration (`undoLastMigration()`), not a range. In docker: `docker compose run --rm n8n db:revert`.
2. It **refuses on irreversible migrations**: "Cancelled command. The last migration `<name>` was irreversible." There is then no supported path back.
3. It must be run **from the new version, before swapping the image**. Its own error text says so: "This usually means that you downgraded n8n before running `n8n db:revert`. Please upgrade n8n again and run `n8n db:revert` and then downgrade again." The old image does not contain the newer migration's code and cannot revert it.
4. **Many migrations have no `down()` at all.** Counted across the migration directories: `postgresdb/` has 57 files — 34 reversible, **16 explicitly `IrreversibleMigration`**, 7 older ones with no `down`; `common/` has 192 files — 166 reversible, **25 irreversible**. Some nominally reversible ones are lossy by design (one comments that its down migration "will reset any count values exceeding INTEGER max … to 0").
5. Docs procedure ([docs][revert]): install the older version, check release notes for manual changes, "Run `n8n db:revert` on your current version to roll back the database. **If you want to revert more than one database migration, you need to repeat this process.**" There is no Docker-page equivalent of this section.

**There is no general documented statement that downgrade is unsupported — and no supported multi-version downgrade mechanism either.** A Postgres dump of n8n's database taken immediately before every upgrade is the only reliable rollback. In a shared engine, take a per-database dump (`pg_dump -d ${N8N_DB_NAME}`), not a cluster-wide one.

---

## 12. Container port and the traefik label

- The container listens on **5678/tcp**. `docker/images/n8n/Dockerfile` line 52: `EXPOSE 5678/tcp` ([source][dockerfile]). Docs confirm `N8N_PORT` default `5678` ([docs][deploy-env]).
- **`N8N_PORT` does change it** — it is the real listen port, not just metadata. `N8N_LISTEN_ADDRESS` (default `::`) sets the bind address, which already accepts connections from the docker network.
- **If you change `N8N_PORT`, the traefik label must follow.** `EXPOSE` is documentation-only and traefik does not read it; `loadbalancer.server.port` must equal the actual listen port.
- Simplest correct choice: leave `N8N_PORT` unset and label `traefik.http.services.n8n.loadbalancer.server.port=5678`.
- Do not expose `5679` (the task-runner broker) through traefik.

---

## 13. Image and architecture

**Confirmed: the official image publishes `linux/amd64`.**

`docker manifest inspect docker.n8n.io/n8nio/n8n:latest` returns an OCI image index containing **linux/amd64** and **linux/arm64** (plus `unknown/unknown` attestation/provenance entries, which are not runnable platforms). The Docker Hub API reports the same pair for `latest`, `stable`, `next`, `beta`, `2.33.7` and `1.123.69`. Per-architecture suffixed tags (`2.33.7-amd64`) are also published but are not needed — the multi-arch index resolves correctly on its own.

**Image reference to use:**

```
docker.n8n.io/n8nio/n8n:2.33.7
docker.n8n.io/n8nio/runners:2.33.7      # only in external runner mode; tag must match
```

`docker.n8n.io/n8nio/n8n` is the reference the docs use ([docs][docker-install]); `n8nio/n8n` on Docker Hub and `ghcr.io/n8n-io/n8n` are the same artefact from the same promotion workflow.

---

## 14. One-way doors

Ordered by how expensive they are to reverse.

1. **Losing `N8N_ENCRYPTION_KEY`.** Every credential in Postgres becomes permanently undecryptable. Not derivable, not stored in the database, not recoverable from a database backup alone. Back the key up separately from the database, and never let n8n auto-generate it into a volume you might delete. (§6)
2. **Enabling `N8N_ENV_FEAT_ENCRYPTION_KEY_ROTATION`.** Docs: turning it back off "makes all data encrypted after you enabled the feature permanently inaccessible", and "The only recovery path is restoring from a database backup taken before you enabled the feature." Do not enable it speculatively. It also blocks downgrading, since older versions cannot decrypt the new format. (§6)
3. **Running a newer n8n against the database.** Migrations run automatically and fatally at startup (§3.2), roughly a third of them have no `down()`, and `db:revert` undoes only one at a time and only from the new image. Rolling back a major is, in practice, restore-from-dump. **Take a `pg_dump` of n8n's database immediately before every image bump**, and never let a channel tag do this unattended. (§11)
4. **Dropping MySQL/MariaDB, and unsupported derivatives.** v2.0 removed MySQL/MariaDB support, and Aurora/AlloyDB/CockroachDB/YugabyteDB are explicitly unsupported. Committing n8n's data to an unsupported engine is a migration project to undo. Not a live risk here — a stock Postgres engine is the right call. (§1.3)
5. **Choosing the binary-data mode after data exists.** Switching `N8N_DEFAULT_BINARY_DATA_MODE` does not migrate existing data, and pruning only reclaims the currently active mode — so old data is orphaned and never cleaned up. Decide `filesystem` against `s3` before there is volume. The same applies to `N8N_EXECUTION_DATA_STORAGE_MODE`. (§7, §9)
6. **`DB_TABLE_PREFIX` and `DB_POSTGRESDB_SCHEMA`.** Both are baked into every table's identity at first migration. Changing either afterwards makes n8n look at an empty schema and migrate a fresh one from scratch; the old tables are simply abandoned. Note the migration advisory lock is keyed on `sha256(schema + prefix)`, so a change also silently moves the lock. (§3.1, §3.4)
7. **Not pruning from day one.** `EXECUTIONS_DATA_PRUNE` defaults to `true`, so this only bites if someone disables it or sets `EXECUTIONS_DATA_PRUNE_MAX_COUNT=0`. Recoverable, but reclaiming space from a bloated shared engine means a `VACUUM FULL` — an exclusive lock on n8n's tables inside a database other tenants share. (§9)
8. **The pgbouncer pool mode itself** — soft, but engine-wide. `pool_mode` can be set per-database in pgbouncer, so if the transaction door ever proves wrong for n8n, moving that one tenant to the session door is a pgbouncer config change and a restart, not a data migration. Keep it per-database rather than global so this stays cheap. (§2)

---

## What could not be established from a primary source

- Whether any of the 57 Postgres migration files depends on `search_path` rather than schema-qualified names (would need a per-file audit). Moot while `DB_POSTGRESDB_SCHEMA=public`.
- Whether n8n's CI or n8n Cloud tests against any connection pooler. No workflow or documentation evidence. Transaction-mode support rests on source reading plus a maintainer's statement in [#26735][issue-26735], not a published test matrix.
- Whether skipping a major version during an upgrade is supported. No documented prohibition and no documented requirement to step through each major.
- Which n8n version introduced encryption-key rotation. The docs describe the feature and its flag but name no version.
- The exhaustive list of consumers of the computed `baseUrl` beyond the webhook/editor/OAuth/email chain traced in §4.
- Whether n8n 2.11.0 (March 2026) genuinely sent `statement_timeout` in the startup packet as [#26735][issue-26735] claims. Current source does not; the historical driver was not diffed.

---

## Source index

[db-env]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/database
[deploy-env]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/deployment
[endpoints-env]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/endpoints
[exec-env]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/executions
[binary-env]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/binary-data
[runners-env]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/task-runners
[tz-env]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/timezone-and-localization
[choose-db]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/choose-n8ns-database
[docker-install]: https://docs.n8n.io/deploy/host-n8n/install-options/install-with-docker
[webhook-proxy]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/configuration-examples/configure-webhook-urls-with-reverse-proxy
[enc-key]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/configuration-examples/set-a-custom-encryption-key
[rotate-keys]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/security/rotate-encryption-keys
[tz-example]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/configuration-examples/set-the-timezone
[runners-setup]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/set-up-task-runners
[binary-scaling]: https://docs.n8n.io/deploy/host-n8n/configure-n8n/scaling/handle-binary-data
[update-n8n]: https://docs.n8n.io/deploy/host-n8n/keep-n8n-running/update-n8n
[v2-breaking]: https://docs.n8n.io/changelog/v20-breaking-changes
[revert]: https://docs.n8n.io/deploy/host-n8n/install-options/install-with-npm#reverting-an-upgrade

[db-config]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/config/src/configs/database.config.ts
[db-conn-options]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/db/src/connection/db-connection-options.ts
[db-connection]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/db/src/connection/db-connection.ts
[db-lock]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/db/src/services/db-lock.service.ts
[db-monitor]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/db/src/connection/db-connection-monitor.ts
[pg-driver]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/typeorm/src/driver/postgres/PostgresDriver.ts
[pg-runner]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/typeorm/src/driver/postgres/PostgresQueryRunner.ts
[pg-trigger]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/nodes-base/nodes/Postgres/PostgresTrigger.functions.ts
[url-svc]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/cli/src/services/url.service.ts
[proxy-hops]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/cli/src/abstract-server.ts
[base-command]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/cli/src/commands/base-command.ts
[instance-settings]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/core/src/instance-settings/instance-settings.ts
[instance-settings-config]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/config/src/configs/instance-settings-config.ts
[binary-config]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/core/src/binary-data/binary-data.config.ts
[exec-config]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/config/src/configs/executions.config.ts
[runners-config]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/config/src/configs/runners.config.ts
[auth-config]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/packages/%40n8n/config/src/configs/auth.config.ts
[dockerfile]: https://github.com/n8n-io/n8n/blob/n8n%402.33.7/docker/images/n8n/Dockerfile
[release-workflow]: https://github.com/n8n-io/n8n/blob/master/.github/workflows/release-push-to-channel.yml
[issue-26735]: https://github.com/n8n-io/n8n/issues/26735
[issue-30612]: https://github.com/n8n-io/n8n/issues/30612
[pr-31008]: https://github.com/n8n-io/n8n/pull/31008

[npg-queries]: https://node-postgres.com/features/queries
[pgb-config]: https://www.pgbouncer.org/config.html
[pgb-features]: https://www.pgbouncer.org/features.html
