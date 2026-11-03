# Run Lens

Lens records agent traces, investigates failures, and turns useful examples into datasets. The standalone deployment includes the UI, API and background processing in one Lens service, with ClickHouse storing its data and coordination state

The source installation below builds the current checkout. A published standalone image and release bundle are still being qualified; this guide does not imply that an installable release has shipped

Prefer your coding agent to make the changes? [Set it up for me](../../docs/setup-with-agent.md#start-standalone-lens) has a copyable prompt for standalone setup, an existing LiteLLM deployment, and optional external storage. The commands below remain the shortest manual path

## Start locally

Install Git and Docker with Docker Compose v2, then run:

```sh
git clone https://github.com/BerriAI/lens.git
cd lens
./deploy/lens/start
```

The first start builds Lens and the ClickHouse image with its Keeper configuration, generates private credentials in `deploy/lens/.env`, and starts Lens plus ClickHouse. You do not need a LiteLLM gateway, PostgreSQL, host Python, Node.js, or provider credentials to record and inspect traces

Open [Lens](http://localhost:4318/ui/) and sign in using `LENS_ADMIN_TOKEN` from `deploy/lens/.env`. Read that file locally; do not paste it into shared logs or commit it. The starter binds Lens to localhost and leaves ClickHouse off the host network

Rerunning the start command preserves the environment file, credentials and ClickHouse volume. The initial image build takes longer than subsequent starts

The `deploy/lens/ctl` wrapper runs Docker Compose with the repository root as the project directory and loads the local override. Use it for the maintenance commands in this guide

## Deploy on Coolify

Coolify runs Lens from the same `deploy/lens/compose.yaml` used locally. Local commands add `deploy/lens/compose.local.yaml` to publish the host port and read `deploy/lens/.env`; Coolify uses only the base file. The base file reads Coolify generated values for the administrator token, the ClickHouse password and the public URL, so a new deployment needs no copied credentials. Coolify also builds the bundled ClickHouse image from `deploy/clickhouse/Dockerfile`, which bakes in the Keeper and logging configuration

Create a project and environment, add a new resource, choose the Git repository and the **Docker Compose** build pack. Use the Git-based Compose application, not a pasted Compose definition, because both services build from this repository. Set **Base Directory** to `/` and **Docker Compose Location** to `/deploy/lens/compose.yaml`. Leave **Preserve repository during deployment** off; the Compose file mounts no repository files

Optional: add a domain for the `lens` service with internal port `4318`, for example `https://lens.example.com:4318`. Use a hostname only; Lens uses the public origin and rejects a URL with a path. Without a domain, Coolify assigns one from its wildcard domain or sslip.io; set a custom domain for production. The public request uses the normal HTTP or HTTPS port; the `:4318` suffix only tells the proxy which container port to reach

Deploy. The first build compiles the Rust service and the UI and pulls the ClickHouse image, so allow a long build and at least 4 GB of memory on the build host. Later deploys reuse the build cache. Size the runtime server for ClickHouse storage plus the Lens memory limit of 2 GB; ClickHouse has no memory limit in this Compose file. If you run more than one Lens resource on the same server, set `LENS_IMAGE` to a unique tag in each resource, for example `lens-production:local`, so their images do not replace one another

When the deployment is healthy, open `https://<your domain>/ui/` and sign in with the value of `SERVICE_PASSWORD_64_LENSADMIN` from the resource's **Environment Variables**. That generated value is the administrator token. Keep it private. Lens refuses to start when it is empty

Coolify stores ClickHouse data in a managed volume named after the application. Removing the resource, or deleting its volumes, deletes traces, investigations and datasets. Back up the volume before destructive changes. The `deploy/lens/backup` helper targets a local checkout, not a Coolify resource; for Coolify, stop the `clickhouse` service during the snapshot or use your volume snapshot tooling, and keep the resource's environment variables with the backup

Optional settings go in the resource's **Environment Variables**. Redeploy after changing them. Trace retention uses `AGENT_TRACING_RETENTION_DAYS`. Provider credentials such as `ANTHROPIC_API_KEY` follow [Configure analysis models](../../docs/analysis.md). To connect a LiteLLM gateway, set `LITELLM_LENS_SERVICE_TOKEN` and `LENS_GATEWAY_SECRET` here with the same values as the gateway, and set `LITELLM_LENS_URL` on the gateway to the Lens service URL. Do not commit these values to the repository

The Lens container keeps its read-only root filesystem, dropped capabilities and no-new-privileges setting. Investigation calculations need a Linux host with Landlock ABI 3 and seccomp, which supported Coolify Docker hosts provide. Use the resource's **Runtime Logs** and the `/health/ready` endpoint when a deploy fails

Coolify applies the Compose health checks, so it waits for ClickHouse and Lens to become healthy. A deployment that stays unhealthy usually means ClickHouse could not start, Lens could not reach ClickHouse, or the public URL does not match the browser origin

## Record your first trace

Open **Home** and choose **Copy instructions with new key**. Lens creates a tracing key and copies complete setup instructions, including your Lens address and key. Paste them into your coding agent inside the project you want to connect. The instructions tell the coding agent to save the key in your project's ignored environment file. The key will also be in that coding agent's chat history. To keep the key out of chat, choose **Set up manually**, create a key, and save it locally as `LITELLM_TRACING_KEY`. You can then return to **Use a coding agent** and choose **Copy without key**

If your project already has a key, choose **Use an existing key**. **Manage tracing keys** lets you select an existing key to watch for its new traces or revoke a key. Existing secret values cannot be retrieved again. Keep your agent's model endpoint and provider credential; tracing keys authorize telemetry uploads and delivery checks

Run one actual agent task. The coding agent verifies that exact trace arrived and gives you a link. Home watches for new traces sent with your selected key and offers **Open trace** when one arrives. With a shared existing key, check that the displayed agent is the one you just connected. For manual configuration, choose **Set up manually**. The **Demo data** switch shows examples; it does not verify your agent connection

For an OTLP exporter, send traces to `http://localhost:4318/v1/traces` with `Authorization: Bearer <tracing-key>`. That address works for an agent running on the same host. An agent in another container or on another machine needs a Lens address reachable from there

You can record traces without an analysis model. When you want Lens to investigate them, follow [Configure analysis models](../../docs/analysis.md), then open **Investigations** and create an investigation with an analysis model and monthly spending limit. [Signals](../../docs/signals.md) use a separate evaluation provider to flag matching traces

## Trace content and size

Lens preserves accepted input, output, and attribute values without truncation. Oversized requests are rejected with HTTP 413. Trace uploads and internal spend batches accept up to 32 MiB per request; gzip uploads must also expand to at most 32 MiB across all members. Other API request bodies, including dataset, eval, signal and feedback requests, accept up to 2 MiB. Reduce the exporter batch size or split an API operation when it exceeds these limits

OTLP decoding defaults to 16,384 spans per upload, 4,096 attributes, events or links per applicable record, 1,000,000 structural nodes and 64 MiB of decoded span allocations. Configure these through `OTLP_MAX_SPANS`, `OTLP_MAX_ATTRIBUTES`, `OTLP_MAX_EVENTS`, `OTLP_MAX_LINKS`, `OTLP_MAX_DECODE_NODES` and `OTLP_MAX_DECODED_SPAN_BYTES`. `OTLP_MAX_DECODE_DEPTH` can lower the nesting limit, but it cannot exceed the absolute ceiling of 64. Existing `CLICKHOUSE_TRACE_MAX_INSERT_BYTES`, `LENS_DATASET_MAX_CASES` and `LENS_DATASET_MAX_CASE_CHARS` settings also remain available

Database reads buffer at most 64 MiB per response. Ad-hoc SQL readers also enforce 50,000 result rows and 64 MiB of result data in ClickHouse. Narrow the time window, select fewer columns or add a smaller `LIMIT` when a query exceeds these limits. Large traces are read in pages; cache eviction does not remove stored traces. Eval runs have no fixed case or trial-count ceiling, so configure spending controls on the model credential used for scoring

Your exporter, reverse proxy, database, and available memory can still constrain the data Lens receives or processes. Model context windows and the [calculation sandbox](../../docs/sandbox.md) also apply when analyzing traces. Content truncated before it reaches Lens, or stored by an older version with truncation, cannot be reconstructed

## Configure a deployment

For Kubernetes, use the [Lens Helm chart](../../helm/lens/README.md). To move an existing gateway-hosted Lens installation and its saved records, follow [Migrate existing Lens data](../../docs/migration.md) before changing which service owns its writers

The bundled deployment uses one ClickHouse server with Keeper. External storage supports that same topology: one stable endpoint reaching one ClickHouse server, shared by any Lens replicas. Lens payload tables are local to that server; Keeper does not replicate them between ClickHouse servers. Load balancing across ClickHouse nodes, distributed tables and automatic failover to a different ClickHouse server are unsupported. Use the [external ClickHouse settings](../../helm/lens/README.md#use-existing-clickhouse) for the required capabilities and permissions

Keep `deploy/lens/.env` private and retain it alongside your persistent data. The generated secrets have different purposes:

| Setting | Used by |
| --- | --- |
| `LENS_ADMIN_TOKEN` | The administrator signing into Lens and authorized API clients |
| `CLICKHOUSE_PASSWORD` | Lens connecting to its bundled ClickHouse server |
| A tracing key generated in Lens | An agent or exporter uploading traces |
| An analysis or evaluation provider key | Lens sending selected trace content to the configured provider |

On Coolify the first two values are generated as `SERVICE_PASSWORD_64_LENSADMIN` and `SERVICE_PASSWORD_64_LENSCH` in the resource's Environment Variables. The Compose file maps them to `LENS_ADMIN_TOKEN` and `CLICKHOUSE_PASSWORD`

For a server deployment outside Coolify, put Lens behind your TLS reverse proxy and set `LENS_PUBLIC_URL=https://lens.example.com` in the environment file. Replace that hostname with yours. This value controls the address displayed to agents and enables secure session cookies. Forward the UI and API paths to port 4318; never expose ClickHouse publicly

If your reverse proxy runs outside this host's network namespace, set `LENS_BIND_ADDRESS` to the private interface it can reach. For a different host port, set `LENS_PORT` and update `LENS_PUBLIC_URL` to the corresponding public address. The default bind address is `127.0.0.1`

Apply environment changes without rebuilding the image:

```sh
./deploy/lens/ctl up -d --wait
```

Provider environment variables in `deploy/lens/.env` are passed to the Lens service. Supply production credentials using your platform's secret management where available

The container runs without root or Linux capabilities, with a read-only filesystem and bounded temporary space. Preserve those settings. Investigation calculations require a native Linux kernel with Landlock ABI 3 and seccomp support, as provided by supported Docker Linux hosts. See the [calculation sandbox](../../docs/sandbox.md) for its boundaries and verification command

## Recorded gateway costs

Lens uses the finalized amounts exported by LiteLLM. On the gateway, set `LITELLM_LENS_URL` to the private Lens service URL. Configure the same `LITELLM_LENS_SERVICE_TOKEN` on both services to enable authenticated spend ingestion. Trace credentials and gateway calls must belong to the same authorized user or key and team for their identifiers to match

New gateway records write their recorded cost and call identifiers into compact Lens storage through the existing background export. Lens acknowledges an upload after both request-log and compact writes succeed. Trace browsing reads the compact records without selecting from `spend_logs`; it does not recalculate prices or accept amounts from agent traces. Calls made without a gateway spend record remain unpriced

### Upgrade and recover historical costs

Upgrade every Lens writer and wait for `/health/ready` before backfilling historical costs. Existing matched historical costs become unavailable until their retained request logs are projected into compact storage. Startup does not scan those logs, and trace browsing has no fallback to the raw table

From the matching source checkout, build the operator command:

```sh
cargo build --manifest-path src/worker/Cargo.toml --locked \
  -p litellm-traces-clickhouse --bin backfill_call_costs --release
```

Run it from a host with private ClickHouse access. Set `CLICKHOUSE_URL` through your secret manager to a credential allowed to select `spend_logs` and insert compact Lens cost records. Set `CLICKHOUSE_DATABASE` if it differs from the default `lens`. Pass one exact team ID, inclusive start and exclusive end as UTC Unix milliseconds. An explicit empty team argument, `''`, selects records with no team

```sh
src/worker/target/release/backfill_call_costs \
  'team-id' 1791504000000 1791590400000
```

Each invocation permits at most a 24-hour window, 20 pages of 500 records and five minutes. Each page also has a ten-second query limit, a 4 MiB response limit, a million-row or 64 MiB scan limit, and a 128 MiB query memory limit. A rejected page leaves the last successful cursor usable; choose smaller time windows when scan limits prevent progress. Only call identifiers, ownership, timestamps and recorded amounts are read, with no prompts or response bodies

Save the printed JSON lines. A cursor is printed and flushed only after its page is persisted. If `complete` is false, a timeout occurs, or the command fails, repeat the same team and time window with the last printed cursor as the fourth argument:

```sh
src/worker/target/release/backfill_call_costs \
  'team-id' 1791504000000 1791590400000 '<cursor>'
```

A cursor is bound to its team and window. Replaying a page or the whole window replaces the existing record identity and version without adding its cost twice. Repeat bounded windows for the historical period you need, then verify trace costs before treating coverage as complete. Expired records cannot be recovered. Backfill restores recorded amounts only; spans with incomplete call evidence remain unmatched

Accepted compact records survive service restarts. The gateway exporter retains its bounded memory queue, three delivery attempts and limited shutdown drain. Callbacks lost before successful ingestion are not recovered by this change. Backfill can recover a raw record retained after an incomplete compact write, but cannot recover a record absent from ClickHouse

## Check and restart

From the repository root:

```sh
./deploy/lens/ctl ps
curl --fail http://localhost:4318/health/ready
./deploy/lens/ctl logs --tail=100 lens
./deploy/lens/ctl restart lens
```

`ps` should show both services healthy. The readiness request returns HTTP 200. Verify a stored trace or dataset in the UI after restarting; readiness alone is not a data-recovery test

Stop the installation while retaining its data:

```sh
./deploy/lens/ctl down
```

Keep the `clickhouse_data` volume and environment file. Adding `--volumes` to `down` deletes the database, including stored traces, investigations, datasets and sessions

Use [Back up and restore Lens](../../docs/backup.md) before changing a persistent deployment. The helper retains ClickHouse data, coordination state, credentials, the installed Lens image and the Compose and ClickHouse build files, and restores them into a separate deployment for verification

## Resolve startup problems

| Symptom | Check and action |
| --- | --- |
| Docker is unavailable | Start Docker, verify `docker info`, and rerun `./deploy/lens/start` |
| Port 4318 is in use | Set a free `LENS_PORT` and corresponding `LENS_PUBLIC_URL` in `deploy/lens/.env`, then apply the configuration |
| A required secret is missing | Check the existing environment file. Setup deliberately preserves it rather than rotating credentials behind your back |
| ClickHouse is unhealthy | Run `./deploy/lens/ctl logs --tail=100 clickhouse`; check disk space and the baked Keeper configuration in `deploy/clickhouse/Dockerfile` |
| Lens cannot reach ClickHouse | Check both services in `ps`, their logs, and the retained database credential |
| Changing the ClickHouse password breaks startup | The ClickHouse user was created with the original password on first start. Restore the old value, or rotate the user inside ClickHouse before changing the environment |
| External ClickHouse answers queries but Lens exits before becoming ready | Check the server's KeeperMap and `keeper_map_path_prefix` configuration and the Lens database user's create, insert, read and alter permissions. An analytical query alone does not establish that Lens can initialize its state |
| Setup says no analysis provider is configured | Tracing can still work. Follow the [analysis guide](../../docs/analysis.md) to enable investigations |
| Login works locally but not through the public hostname | Check that `LENS_PUBLIC_URL` matches the browser origin and that your proxy forwards HTTPS correctly |

Official paired Lens and LiteLLM releases share a version and are tested together. Lens can still be installed and operated on its own. Follow the [migration guide](../../docs/migration.md) to transfer existing Lens data
