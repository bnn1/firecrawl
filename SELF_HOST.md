# Self-hosting Firecrawl

Want to get Firecrawl running? Start with the
[Firecrawl self-hosting guide](https://docs.firecrawl.dev/contributing/self-host).
It takes you from checkout to a successful scrape with Docker Compose.

Use this file when you are changing the baseline. It stays with the source, so
the services and configuration match the revision you checked out.

## Pick the guide for the job

| If you need to decide or do this                          | Start here                                                                                                                                                |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Decide whether self-hosting fits and run the first scrape | [Public self-hosting guide](https://docs.firecrawl.dev/contributing/self-host)                                                                            |
| Check which variables and services exist at this revision | [Root Compose configuration](./docker-compose.yaml)                                                                                                       |
| Adapt a Kubernetes deployment                             | [Kubernetes manifests](./examples/kubernetes/cluster-install/) or [Helm chart](./examples/kubernetes/firecrawl-helm/)                                     |
| Change Firecrawl product code                             | [Running Locally](https://docs.firecrawl.dev/contributing/guide), then the [contribution guide](./CONTRIBUTING.md)                                        |
| Connect an agent or terminal client                       | [Local MCP](https://docs.firecrawl.dev/mcp-server/local) or [Firecrawl CLI](https://docs.firecrawl.dev/sdks/cli#connect-the-cli-to-self-hosted-firecrawl) |

## Keep the first run simple

- **Release: an exact tag.** Review the target release's Compose file before
  changing it. A checkout of `main` and floating image tags can change
  independently.
- **API authentication: `USE_DB_AUTHENTICATION=false`.** Add authentication
  after provisioning the required database schema and application
  configuration. Changing this variable alone is not a complete authenticated
  deployment.
- **Queue: NuQ PostgreSQL.** Keep `NUQ_BACKEND=pg` and `FDB_CLUSTER_FILE`
  empty unless you intentionally configure FoundationDB as described below.
- **Scraping: bundled Playwright with basic fetch fallback.** Connect and
  configure a separate engine such as Fire-engine only when you need it.
- **AI-backed features: no model provider.** Connect OpenAI, an OpenAI-compatible
  endpoint, or Ollama when a feature needs it.
- **Queue administration UI: off.** Enable it only with a strong
  `BULL_AUTH_KEY` and restricted network access.

Get this baseline working before swapping backends or adding providers.

The root `.env` overrides only variables referenced by `docker-compose.yaml`.
Do not use `apps/api/.env.example` as a drop-in Compose contract.

## Deploy with Coolify on a VPS

Use a **Git-based application** with the **Docker Compose** build pack, not
Nixpacks or the API Dockerfile alone. The API needs the other Compose services.

1. Select this repository and the branch containing your deployment changes.
   Set **Base Directory** to `/` and **Docker Compose Location** to
   `/docker-compose.yaml`. Reload the Compose definition after updating the
   branch. Leave **Raw Compose Deployment** disabled so Coolify manages proxy
   routing and its Compose extensions.
2. In **Environment Variables**, keep `USE_DB_AUTHENTICATION=false`,
   `NUQ_BACKEND=pg`, and `FDB_CLUSTER_FILE` empty for the first deployment.
   Set a strong `POSTGRES_PASSWORD`; keep `POSTGRES_DB=postgres` for the bundled
   `pg_cron` configuration. Leave the Redis, RabbitMQ, PostgreSQL host, and
   Playwright URLs at their Compose defaults so they use service-name DNS.
   Do not paste `apps/api/.env.example` into Coolify: it is not the Compose
   environment contract. An OpenAI key is not required for ordinary scraping.
3. The defaults target a shared 4-vCPU / 8-GiB VPS. The five services have a
   combined memory ceiling of **4.75 GiB**, leaving roughly 3.25 GiB of an
   8-GiB machine for the OS, Coolify, and other services (actual usable RAM
   varies). Limits are ceilings, not reservations, and do not limit image
   builds. Large pages or documents can still exhaust a container's limit.
   See the resource budget below.
4. For domain access, assign a domain **only to `api`**, for example
   `https://crawl.example.com:3002`. Point its DNS record at the VPS. The port
   suffix tells Coolify to route to container port `3002`; clients still use
   `https://crawl.example.com` on normal HTTPS port `443`. If you change
   `INTERNAL_PORT`, use that port in the domain setting too. Do not assign
   domains or publish ports for the databases, browser, or workers.
5. Before making the domain public, restrict access with an authenticated
   reverse proxy or a private network/VPN. With `USE_DB_AUTHENTICATION=false`,
   the API does not validate API keys. Neither `TEST_API_KEY` nor
   `BULL_AUTH_KEY` adds API authentication. HTTPS alone is not access control.
6. Deploy. The API health check requests `/` inside the container. A successful
   response confirms HTTP availability, not a successful scrape. Check the API
   and worker logs, then run the scrape below.

The host mapping defaults to **`172.30.0.3:3002`**, VPS 1's Hetzner private
address. From VPS 2 (`172.30.0.2`), use **`http://172.30.0.3:3002`** as the
Firecrawl base URL. No public domain or public port opening is required.
VPS 1 must have this private address attached before deployment. For another
host, set `API_BIND_ADDRESS` to its private IP; for local-only access, set it
to `127.0.0.1`. Do not set it to `0.0.0.0` on a public VPS.

Binding to a private IP does not restrict callers to VPS 2 alone. The default
API is unauthenticated: trust the attached private-network members, or enforce
a Docker-aware host firewall policy allowing TCP port `3002` from
`172.30.0.2/32` and rejecting other sources. Ordinary UFW rules may be bypassed
by Docker-published ports. Private Hetzner traffic is not automatically
encrypted; use TLS if required. Remove any existing Coolify public domain or
proxy route if this deployment should be private-only.

From either VPS, verify the API and a non-AI scrape:

```bash
curl --fail-with-body http://172.30.0.3:3002/

curl --fail-with-body http://172.30.0.3:3002/v2/scrape \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://example.com","formats":["markdown"]}'
```

Use the configured host `PORT` instead of `3002` if you changed it. For a
domain request, use your HTTPS URL and the credentials required by your access
control layer.

### Shared-host resource budget

| Service                      | Memory ceiling | CPU ceiling |
| ---------------------------- | -------------- | ----------- |
| API and its workers combined | 2560 MiB       | 2           |
| Playwright                   | 1024 MiB       | 1           |
| NuQ PostgreSQL               | 512 MiB        | 0.5         |
| RabbitMQ                     | 512 MiB        | 0.5         |
| Redis                        | 256 MiB        | 0.25        |

Container swap is disabled by setting each `memswap_limit` equal to its memory
limit. CPU ceilings are per-service maximums, not reserved cores.
The browser's temporary cache is capped at 256 MiB and counts toward its
container memory ceiling. Redis caps its dataset at 192 MiB with `noeviction`:
when full it rejects writes rather than evicting queue or lock data. Monitor
memory use and failed jobs; this is a low-throughput baseline, not a guarantee
for arbitrary workloads.

RabbitMQ has a 90-second startup grace period and a 15-second health-check
timeout for this CPU-limited configuration. A successful check allows the API
to start immediately; it does not have to wait the full grace period.
The API still requires RabbitMQ to be healthy before starting.

The effective scrape-process control is **`NUQ_WORKER_COUNT=1`**.
**`CRAWL_CONCURRENT_REQUESTS=2`** sets the bundled browser's page limit.
The old Compose variables `NUM_WORKERS_PER_QUEUE`, `MAX_CONCURRENT_JOBS`, and
`BROWSER_POOL_SIZE` were unused by this revision and have been removed.
In Coolify, clear or update previously saved concurrency values: Compose
defaults do not overwrite existing environment values.

FoundationDB and its initializer are behind the `fdb` Compose profile, so they
do not run in the default PostgreSQL deployment. If an older deployment left
them running, stop those two services through Coolify; do not delete their
volumes. Keep that profile disabled on this shared-host baseline.

### Coolify parsing and initialization

Coolify's environment parser does not support Compose's `:+` alternative-value
operator. An expression such as `${NUQ_BACKEND:+/var/fdb/fdb.cluster}` is
interpreted as an invalid variable name and produces **“The key must start with
a letter or underscore…”** before any image is built. The root Compose file
uses a separate, optional `${FDB_CLUSTER_FILE:-}` reference instead.

Coolify retains previously saved environment values when reloading Compose.
Clear `FDB_CLUSTER_FILE` for PostgreSQL-only deployments.

The `foundationdb-init` container is a one-shot job: exiting successfully is
expected. Its `exclude_from_hc: true` flag tells Coolify not to count that exit
against application health. This is a **Coolify extension**, not standard
Compose. For standalone `docker compose` use, omit only that flag in a local
copy; keep it in the definition deployed through Coolify.

See [Coolify's Docker Compose documentation](https://coolify.io/docs/applications/builds/docker-compose)
for domain routing, environment variables, and health-check behavior.

### Optional FoundationDB queue

Only when intentionally using the experimental FoundationDB backend, set:

```dotenv
COMPOSE_PROFILES=fdb
NUQ_BACKEND=fdb
FDB_CLUSTER_FILE=/var/fdb/fdb.cluster
```

The deployment's Compose invocation must enable the `fdb` profile (for example
`docker compose --profile fdb up -d`, or `COMPOSE_PROFILES=fdb` in its
environment). Merely passing the variable inside the API container does not
enable a Compose profile. Budget additional RAM for FoundationDB; the
4.75-GiB ceiling above covers only the default PostgreSQL deployment.

The path is inside the API container's shared volume, not a path on the VPS.
Wait for `foundationdb-init` to exit with code `0` before sending work to that
backend. Keep the cluster-file variable empty in PostgreSQL mode: a nonempty
value also enables optional FoundationDB lookups in the queue router.

## What the stack runs

By default, Compose runs the Firecrawl API and workers, Playwright, Redis,
RabbitMQ, and NuQ PostgreSQL. FoundationDB services require the `fdb` profile.
Only the API is published to the host, on `172.30.0.3:3002` by default;
Coolify can still route domain traffic over the Docker network if configured.

Self-hosting gives you source and infrastructure control. You also own
security, availability, capacity, upgrades, data retention, and compliance.

## Before production

- **If the API will leave a trusted network,** add a complete authentication
  design, TLS termination, and network policy first. The default API is
  unauthenticated.
- **If data must survive service replacement,** add and test persistence,
  backups, and recovery for NuQ PostgreSQL, Redis, and RabbitMQ. The root
  Compose file defines no persistent volumes for them.
- **If you change the PostgreSQL settings,** keep the API and database values
  consistent. At this revision, the bundled `pg_cron` configuration targets
  the default `postgres` database.
- **If you publish dependency ports,** secure them explicitly. PostgreSQL,
  Redis, RabbitMQ, and worker ports should remain private by default.
- **If you have availability or scale targets,** define monitoring, resource
  limits, scaling triggers, and upgrade and rollback procedures. The checked-in
  Compose file is a source-aligned starting point, not a production
  architecture.

Treat the Kubernetes and Helm examples as versioned starting points, not as
evidence that these production decisions have been made for you.

Stuck? Open a
[self-host issue template](https://github.com/firecrawl/firecrawl/issues/new?template=self_host_issue.md)
or join the [Firecrawl Discord community](https://discord.gg/firecrawl).
