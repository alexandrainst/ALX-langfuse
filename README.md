# Langfuse (self-hosted, standalone)

Langfuse v3 as its own Compose stack.
One instance can serve several projects, each with its own API key pair.

## Layout

| File | Purpose |
| --- | --- |
| `docker-compose.yml` | The stack: web, worker, postgres, clickhouse, redis, minio |
| `example.env` | Template — copy to `.env` |
| `.env` | Real config (gitignored) |

## First run

```bash
cd .devcontainer/langfuse
cp example.env .env      # then fill in the secrets
docker compose up -d
```

UI at http://localhost:3000 (or `LANGFUSE_WEB_PORT`).

Either stack can start first: both declare `langfuse_net` by name, so whichever
comes up first creates it and the other joins it.

## Start / stop

```bash
docker compose up -d     # start
docker compose stop      # stop, keep data
docker compose down      # remove containers, keep data (volumes are named)
docker compose down -v   # DESTROY all traces and config
```

The backend tolerates Langfuse being down — tracing is fire-and-forget, so
requests still serve. Stopping this stack whenever it isn't needed is safe.

## Connecting a project

A consumer needs two things in its own env:

```sh
LANGFUSE_HOST='http://langfuse-web:3000'   # in-network DNS name
LANGFUSE_PUBLIC_KEY='pk-lf-...'
LANGFUSE_SECRET_KEY='sk-lf-...'
```

and its Compose file must join the network:

```yaml
networks:
  langfuse_net:
    name: langfuse_net
```

`LANGFUSE_HOST` is the container name, resolvable only from inside
`langfuse_net`. Browsers use `LANGFUSE_BASE_URL` instead.

The shared network is a local-dev convenience. A Docker network cannot span
hosts, so a deployed Langfuse is reached over its URL instead
(`LANGFUSE_HOST=https://...`) with no network coupling and no startup order.

## Keys

`LANGFUSE_INIT_*` in `.env` applies **only while the Postgres volume is empty**.
It seeds the first org, user and project so a fresh instance has an account to
log in with. On every later start it is inert — it does not re-seed or reconcile
an existing instance, and editing it changes nothing.

Every project's API keys come from the UI: open the project, then Settings → API
Keys, and copy the pair into that project's own env. This stack holds no
consumer keys.

## Data

Traces live in **ClickHouse**; Postgres holds orgs, projects and users. All five volumes are declared with an explicit `name:`, so their identity does not depend on the Compose project name or the directory the stack was launched from, and data survives `down` / `up` and rebuilds.

Prefer `docker compose stop`/`down` over killing the containers: ClickHouse
buffers recent inserts in memory, and an abrupt stop can drop the newest traces.

## Troubleshooting

**Do not quote values in `.env`.** Compose does not strip quotes when
interpolating them into `docker-compose.yml`, so `LANGFUSE_WEB_PORT='3000'`
yields the invalid port spec `'3000':3000`, which is dropped silently.

**`docker ps` shows `3000/tcp` instead of `0.0.0.0:3000->3000/tcp`.** The port
mapping was never applied. Mappings are fixed when a container is created, so
`start` and `restart` cannot add one — recreate it:

```bash
docker compose up -d --force-recreate --no-deps langfuse-web
```

**`langfuse-web` / `langfuse-worker` stay in `Created` with empty logs.** They
were never started. Check the datastores are healthy (`docker compose ps`); if
they are, the daemon stalled — start the container directly to see the real
error, and restart Docker Desktop if that hangs:

```bash
docker start langfuse-langfuse-web-1
docker logs langfuse-langfuse-web-1 --tail 30
```

**`curl http://langfuse-web:3000` fails from the host.** Expected: that name
resolves only inside `langfuse_net`. Use `localhost:3000` from the host, and the
container name only from inside a container on that network.
