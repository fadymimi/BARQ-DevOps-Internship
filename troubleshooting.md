# Troubleshooting journal

Keep chronological entries. Copy this block for each meaningful investigation.

## Entry 1 / 2026-09-13 / initial environment start
- Symptom: README instructs `docker compose -p barq-assessment up --build -d`. Ran it; build succeeded but the stated behavior ("not expected to pass") needed verification.
- Hypothesis: Something in the Compose/NGINX/env layer is broken while app logic (server.py) matches the API contract on read-through.
- Command or test: `git log -2 --oneline`, `cp .env.example .env`, `docker compose -p barq-assessment up --build -d`, `docker compose -p barq-assessment ps -a`
- Actual output: All 5 containers started. `app-01` and `app-02` show `Up (unhealthy)`. `postgres` and `redis` show `Up (healthy)`. `nginx` shows `Up`, port mapping `127.0.0.1:8080->81/tcp`.
- Failed attempt and what changed your thinking: None yet at this stage — this was the baseline observation step.
- Root cause: Not yet isolated; multiple independent issues suspected (see following entries).
- Fix: Pending — see Part 2 commits.
- Retest evidence: Pending.
- Related commit: (baseline observation, no fix yet)
- Remaining uncertainty: Whether app-01/app-02 have additional issues beyond healthcheck once other layers are fixed.

## Entry 2 / 2026-09-13 / healthcheck path mismatch
- Symptom: `app-01`/`app-02` reported `(unhealthy)` in `docker compose ps -a`.
- Hypothesis: The Compose healthcheck calls a path that doesn't exist on the Flask app.
- Command or test: Compared `docker-compose.yml` healthcheck (`.../healthz`) against `app/server.py` routes (`@app.get("/health")`); confirmed via `docker compose logs`.
- Actual output: Repeated log lines: `"path": "/healthz", "status": 404` every ~5 seconds from both app-01 and app-02.
- Failed attempt and what changed your thinking: N/A — confirmed directly from logs on first check.
- Root cause: Compose healthcheck targets `/healthz` (with a trailing z); the app only implements `/health`. Any 404 makes Docker mark the container unhealthy.
- Fix: Pending — will change healthcheck URL to `/health` in Part 2.
- Retest evidence: Pending.
- Related commit: (to be added in Part 2)
- Remaining uncertainty: None — root cause is unambiguous from the exact string mismatch.

## Entry 3 / 2026-09-13 / app binds to loopback only
- Symptom: Requests through NGINX are expected to reach app-01/app-02, but this needs confirming apps are reachable from outside their own container.
- Hypothesis: `APP_HOST=127.0.0.1` in docker-compose.yml makes Flask listen only on the container's loopback interface, unreachable from other containers (e.g. NGINX) on the same network.
- Command or test: Read `docker-compose.yml` x-app-env block (`APP_HOST: "127.0.0.1"`) and app-01 log line.
- Actual output: `app-01  |  * Running on http://127.0.0.1:8080`
- Failed attempt and what changed your thinking: N/A — confirmed by direct log inspection.
- Root cause: Binding to 127.0.0.1 inside a container only accepts connections originating from within that same container's network namespace; NGINX (a separate container) cannot reach it.
- Fix: Pending — will change APP_HOST to 0.0.0.0 in Part 2.
- Retest evidence: Pending.
- Related commit: (to be added in Part 2)
- Remaining uncertainty: None.

## Entry 4 / 2026-09-13 / NGINX host port does not match container listen port
- Symptom: `curl http://127.0.0.1:8080/` from the VM host returned `curl: (56) Recv failure: Connection reset by peer`.
- Hypothesis: docker-compose.yml maps host port 8080 to container port 81, but nginx.conf has `listen 80;`, so nothing is listening on port 81 inside the nginx container.
- Command or test: `curl -i http://127.0.0.1:8080/`; cross-checked `ports: ["127.0.0.1:${PUBLIC_PORT:-8080}:81"]` against `listen 80;` in nginx/nginx.conf.
- Actual output: `curl: (56) Recv failure: Connection reset by peer` (both `/` and `/ready` attempts).
- Failed attempt and what changed your thinking: N/A — port mismatch was visible directly by comparing the two config files.
- Root cause: Compose publishes host:8080 to container-port:81, but NGINX process listens on port 80 inside the container. Nothing answers on 81.
- Fix: Pending — will align the container-side port in Part 2 (either change Compose to map to 80, or change nginx.conf listen to 81 — decision to be recorded in decisions.md).
- Retest evidence: Pending.
- Related commit: (to be added in Part 2)
- Remaining uncertainty: None on the mismatch itself; which side to change is a design decision, not a bug.

## Entry 5 / 2026-09-13 / NGINX upstream ports do not match app listen port
- Symptom: Even once NGINX itself is reachable, requests to the upstream would still be expected to fail.
- Hypothesis: nginx.conf upstream block lists `server app-01:8081` (wrong port) and `server app-02:8080` (correct port, but inconsistent with app-01's entry).
- Command or test: Read nginx/nginx.conf upstream block; compared against APP_PORT="8080" in docker-compose.yml shared environment.
- Actual output: Static config review — not yet tested live via NGINX since NGINX itself is unreachable per Entry 4.
- Failed attempt and what changed your thinking: N/A — found via config read, to be confirmed live once Entry 4's fix is in place.
- Root cause: app-01 upstream entry uses port 8081, but every app instance actually listens on port 8080 (per shared APP_PORT). This is a typo/inconsistency in nginx.conf.
- Fix: Pending — will correct upstream entry to `app-01:8080` in Part 2.
- Retest evidence: Pending.
- Related commit: (to be added in Part 2)
- Remaining uncertainty: Needs live confirmation after Entry 3 and Entry 4 fixes are applied together, since they're interdependent.

## Entry 6 / 2026-09-13 / duplicate INSTANCE_ID for app-01 and app-02
- Symptom: The API contract requires `/instance` to return a distinct instance_id per backend.
- Hypothesis: Both app-01 and app-02 service definitions set INSTANCE_ID to the same literal value.
- Command or test: Read docker-compose.yml service definitions for app-01 and app-02; confirmed via container logs.
- Actual output: app-02's own log line reads `"instance_id": "app-01"` (not "app-02") — confirms the compose file's copy-paste value, not just a display artifact.
- Failed attempt and what changed your thinking: N/A — confirmed directly from the log line itself.
- Root cause: docker-compose.yml sets `INSTANCE_ID: "app-01"` under both the app-01 and app-02 service blocks (copy-paste error).
- Fix: Pending — will set app-02's INSTANCE_ID to "app-02" in Part 2.
- Retest evidence: Pending.
- Related commit: (to be added in Part 2)
- Remaining uncertainty: None.

## Entry 7 / 2026-09-13 / DATABASE_URL and REDIS_URL point at wrong ports and password
- Symptom: `/ready` returns 503 when called directly inside app-01's container.
- Hypothesis: config/app.env has a wrong PostgreSQL password (last character differs from docker-compose.yml's POSTGRES_PASSWORD) and wrong ports for both PostgreSQL (5433 vs actual 5432) and Redis (6380 vs actual 6379).
- Command or test: `docker exec app-01 python3 -c "...urlopen('http://127.0.0.1:8080/ready'...)"`; compared config/app.env values against docker-compose.yml POSTGRES_PASSWORD and the actual listening ports shown in postgres/redis container logs.
- Actual output: `urllib.error.HTTPError: HTTP Error 503: SERVICE UNAVAILABLE`. Postgres log confirms `listening on ... port 5432`; Redis log confirms `port=6379`, while app.env specifies `:5433` and `:6380` respectively, and a password ending in `...7qN2vK8d` vs compose's `...7qN2vK8c`.
- Failed attempt and what changed your thinking: N/A — found by direct value comparison across files plus the live 503 confirming a real (not cosmetic) connection failure.
- Root cause: config/app.env has incorrect port numbers and a mistyped password character for both dependency URLs.
- Fix: Pending — will correct both URLs to match the actual service ports and password in Part 2.
- Retest evidence: Pending.
- Related commit: (to be added in Part 2)
- Remaining uncertainty: None on the specific mismatches; will retest via `/ready` and `/records` after the fix.

## Entry 8 / 2026-09-13 / PostgreSQL volume path and tmpfs override break persistence
- Symptom: Part 3 requires proving a record survives container recreation; this needs the data directory to be a real, non-volatile mount.
- Hypothesis: docker-compose.yml mounts the named volume to `/var/lib/postgresql/backup` (not PostgreSQL's actual data directory) and separately mounts `tmpfs` over `/var/lib/postgresql/data` (PostgreSQL's real data directory), which is RAM-backed and non-persistent.
- Command or test: Read docker-compose.yml postgres service `volumes:` and `tmpfs:` keys.
- Actual output: Static config review — `volumes: postgres-data:/var/lib/postgresql/backup` and `tmpfs: [/var/lib/postgresql/data]`.
- Failed attempt and what changed your thinking: N/A — found via config read; will be confirmed live with an actual persistence test in Part 3.
- Root cause: The named volume is mounted to the wrong path, and a tmpfs mount shadows the real data directory, so all PostgreSQL data lives only in memory and is lost on container removal.
- Fix: Removed the `tmpfs: [/var/lib/postgresql/data]` line and changed the named volume mount from `/var/lib/postgresql/backup` to `/var/lib/postgresql/data` in docker-compose.yml (commit: mount named volume at postgres data path).
- Retest evidence: Created a record titled PERSISTENCE_TEST_RECORD (id 3) via POST /records, ran `docker compose down` (without --volumes) then `docker compose up --build -d`, and confirmed via GET /records that id 3 / PERSISTENCE_TEST_RECORD was still present after full container recreation.
- Related commit: (to be added in Part 2/3)
- Remaining uncertainty: None on the misconfiguration; persistence will be proven empirically once fixed.

## Entry 9 / 2026-09-13 / secrets and prohibited host ports
- Symptom: Task requires "Keep secrets out of images, code and Compose" and "Do not publish app, PostgreSQL or Redis ports."
- Hypothesis: docker-compose.yml hardcodes POSTGRES_PASSWORD in plain text and publishes postgres (15432) and redis (16379) ports to the host.
- Command or test: Read docker-compose.yml postgres/redis service blocks.
- Actual output: `POSTGRES_PASSWORD: BarqLabOnly_7qN2vK8c` inline in the compose file; `ports: ["127.0.0.1:15432:5432"]` and `["127.0.0.1:16379:6379"]` present.
- Failed attempt and what changed your thinking: N/A — direct read of the file.
- Root cause: Compose file violates both the "no secrets in Compose" and "no published DB/cache ports" requirements from the task brief.
- Fix: Pending — will move the password to `.env` (gitignored) referenced via variable substitution, and remove the postgres/redis `ports:` mappings, in Part 2.
- Retest evidence: Pending.
- Related commit: (to be added in Part 2)
- Remaining uncertainty: None.

## Entry 10 / 2026-09-13 / NGINX has direct access to the backend network
- Symptom: Task requires "Block direct NGINX access to PostgreSQL/Redis."
- Hypothesis: nginx service in docker-compose.yml is attached to both `frontend` and `backend` networks, giving it a network path to postgres/redis even though nothing in nginx.conf currently proxies to them.
- Command or test: Read docker-compose.yml nginx service `networks:` key.
- Actual output: `networks: [frontend, backend]` under the nginx service.
- Failed attempt and what changed your thinking: N/A — direct config read.
- Root cause: nginx is unnecessarily joined to the backend network, which the task specifies should be isolated from anything except the apps and the data stores.
- Fix: Pending — will restrict nginx to `frontend` only in Part 2.
- Retest evidence: Pending — will confirm with a connectivity test from inside the nginx container.
- Related commit: (to be added in Part 2)
- Remaining uncertainty: None.
