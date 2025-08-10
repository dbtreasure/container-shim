# PRD — Docker REST API Shim on Apple Containerization (Tahoe)

0) One-liner

Expose a local Docker Engine–compatible REST API that translates to Apple’s Containerization framework so docker and docker compose work on macOS Tahoe without Docker Desktop.

⸻

1) Goals (grug edition)
	•	Keep it tiny. Ship the smallest API surface that runs common Compose v3 stacks (web + db + cache + worker).
	•	Zero re-authoring. Existing docker/docker compose CLIs and dev tools target our socket and “just work” for core flows.
	•	Deterministic mapping. Every Docker object maps cleanly to Apple runtime primitives.

Non-Goals (v1)
	•	Swarm/overlay, plugins, secrets/configs, swarm services.
	•	Full BuildKit/buildx parity (we’ll do simple builds first).
	•	Remote TLS daemon; start local-only (/var/run/docker.sock).
	•	Full docker stats parity (basic later).

⸻

2) Target users / use cases
	•	Devs running Compose v3: FastAPI + Postgres + Redis + workers, Node/Rails + DB, etc.
	•	Editors/IDEs (VS Code devcontainers) that talk Docker API.
	•	CI runners on macOS that need docker build/docker run locally.

⸻

3) Assumptions & constraints
	•	macOS Tahoe (26) with Apple Silicon; Containerization APIs available.
	•	Each container = lightweight VM; each gets its own IP.
	•	We can bind to /var/run/docker.sock.
	•	Swift implementation (SwiftNIO for HTTP), direct framework calls (no shelling out, except temporary escape hatches if needed).

⸻

4) North-star compatibility
	•	Compose v3 (single host): services, images/build, env, ports, volumes, depends_on (ordering/health), restart, healthcheck.
	•	Docker CLI: version, info (minimal), images, pull, build, ps, create, start, stop, rm, logs, exec, inspect, run (via create+start).
	•	Minimal networks/volumes that satisfy Compose semantics.

⸻

5) Phased plan (ruthless scope cut)

Phase 0 — Spike & API skeleton (foundations)
	•	System endpoints: /_ping, /version, /info (static/minimal).
	•	Images: GET /images/json, POST /images/create?fromImage=… (pull), GET /images/{id}/json.
	•	Containers: GET /containers/json, POST /containers/create, POST /containers/{id}/start, POST /containers/{id}/stop, DELETE /containers/{id}, GET /containers/{id}/json.
	•	Mapping: Docker container name → Apple container name; track IDs.
	•	State store (in-proc + on-disk journal): container metadata, Apple IDs, assigned IPs, port proxies, mounts.
	•	Happy-path demo: docker run --name hello nginx:alpine + port publish to localhost via proxy.

DoD: docker run -d -p 8080:80 nginx works; docker ps/logs/stop/rm work.

⸻

Phase 1 — Compose MVP (run common stack)
	•	Networks (MVP):
	•	POST /networks/create, GET /networks, DELETE /networks/{id} → create logical networks (bookkeeping).
	•	All containers actually share Tahoe vmnet; service discovery provided by hosts-file injection per container (/etc/hosts), so names resolve (db, redis).
	•	Volumes (MVP):
	•	POST /volumes/create, GET /volumes, DELETE /volumes/{name} → host dir at ~/.apple-docker/volumes/<name>; mount with --mount type=bind.
	•	Ports:
	•	-p host:container via embedded TCP proxy (SwiftNIO) per mapping (replace socat).
	•	Depends_on / healthchecks:
	•	Ordering: start deps first.
	•	Health: support container HEALTHCHECK probe reading Dockerfile or Compose; implement pollers to mark healthy before starting dependents (timeout/backoff).
	•	Restart policies: support no, on-failure, always (simple supervisor loop).
	•	Logs/exec:
	•	GET /containers/{id}/logs (stream from vsock/pty tap).
	•	POST /containers/{id}/exec + …/start (interactive ok, basic TTY).

DoD: docker compose up runs FastAPI+Postgres+Redis+worker unchanged; down cleans containers, ports, and volumes (if requested).

⸻

Phase 2 — Build & ergonomics
	•	Build: POST /build (context tar stream) → Apple build; stream progress to client. Cache on host path; tags applied.
	•	Auth: POST /auth//images/create with registry creds; store in macOS Keychain.
	•	docker run parity for common flags: -e, -v, --name, --cpus, --memory, --restart, --health-*, --workdir, --entrypoint.

DoD: Can build app images from repo and compose them (no Desktop needed).

⸻

Phase 3 — Observability & resilience
	•	Events: GET /events (limited: start/stop/die/pull/build).
	•	Basic stats: GET /containers/{id}/stats (CPU/mem from VM process; approximate).
	•	Crash recovery: reconcile on start (enumerate Apple containers, rebuild state).
	•	Garbage collection: clean orphaned proxies/mounts on container removal.

⸻

Phase 4 — Networking/volume fidelity (iterative)
	•	DNS stub (tiny resolver) instead of hosts hacks.
	•	Optional per-network subnet emulation (policy routing/NAT) if Tahoe APIs permit.
	•	Volume drivers SPI (later).

⸻

6) Architecture (SOLID, minimal)

Modules (single binary, clear seams):
	•	ApiServer (HTTP/JSON over Unix socket; request routing).
	•	ImageService (pull/list/inspect/build).
	•	ContainerService (spec translation; create/start/stop/rm/exec/logs).
	•	NetworkService (logical networks, name resolution, IP bookkeeping).
	•	VolumeService (named volume registry, host path allocator, mounts).
	•	PortPublisher (SwiftNIO TCP forwarders; lifecycle tied to container).
	•	StateStore (interface; file-backed journal + in-memory index; crash-safe).
	•	AppleRuntime (facade over Containerization APIs; no CLI shell-outs).

Key patterns:
	•	Interfaces per domain (Dependency Inversion): ApiServer depends on service interfaces, not impls.
	•	DTO mappers: Docker API JSON ↔ internal model ↔ Apple model.
	•	Idempotent ops: re-apply desired state safely.
	•	Supervisor per container for restart/health.

⸻

7) Mapping (high-level)

Create container → Apple VM spec
	•	Image ref, env, cwd, entrypoint/cmd, mounts, CPU/mem.
	•	Labels: com.appledocker.project, network=<name>, volumes=<…> for reconciliation.

Start → launch VM
	•	Attach stdout/stderr to a pty/pipe for logs; open vsock for exec.

Port publish
	•	Register <hostIP:hostPort> → <containerIP:containerPort> in PortPublisher.

Networks
	•	Maintain network membership in StateStore.
	•	On container start/attach to networks: update /etc/hosts inside VM with all peers (or DNS stub later).

Volumes
	•	Resolve named volumes → host dirs; validate permissions; mount bind.

⸻

8) API surface by phase (explicit)

P0/P1 must-have
	•	/_ping, /version, /info
	•	/images/json, /images/create, /images/{id}/json
	•	/containers/json, /containers/create, /containers/{id}/json, /containers/{id}/start, /containers/{id}/stop, /containers/{id}/kill, /containers/{id}/wait, /containers/{id}, /containers/{id}/logs, /containers/{id}/attach(optional), /containers/{id}/exec, /exec/{id}/start
	•	/networks/create|list|inspect|delete (logical)
	•	/volumes/create|list|inspect|delete
	•	/build (P2)

Return 501/ignored gracefully for unsupported flags; document.

⸻

9) Compatibility contract (v1)
	•	Works: docker, docker compose (local), VS Code devcontainers.
	•	Diffs (doc’d):
	•	Multiple user networks share same underlying vmnet (isolation best-effort).
	•	stats approximate; some resource flags no-op.
	•	Privileged/devices not supported initially.
	•	BuildKit features limited (no cache-mount/secret-mount v1).

⸻

10) Security
	•	Local Unix socket (user perms); no TCP listener v1.
	•	No root escalation in guest; VM boundary is isolation.
	•	Keychain for registry creds.
	•	Validate bind mounts (block / and sensitive paths by default; allowlist config).

⸻

11) Testing strategy
	•	Conformance: run a curated subset of Docker Engine API tests (where practicable).
	•	Compose suites: sample apps (FastAPI+Postgres+Redis, Rails+Postgres, Node+Mongo).
	•	Soak: start/stop loops; port flaps; crash and reconcile.
	•	Failure modes: image not found, port already in use, volume path invalid, exec on dead container.

⸻

12) Metrics for success
	•	Compose MVP stack: zero file edits; up, logs -f, exec, down pass.
	•	docker run path: p50 container start < 2s; p95 port-open < 2.5s.
	•	Idle CPU ≈ 0%, idle mem < 50MB when no containers.
	•	No orphaned proxies/mounts after rm/down.

⸻

13) Risks & mitigations
	•	Networking fidelity: names may glitch → start with hosts injection; add DNS stub later.
	•	Bind-mount perf: document expectations; perf test common stacks; consider async file I/O hints.
	•	Crash recovery: durable StateStore; reconcile on boot by querying Apple runtime.
	•	API drift: pin Docker API version; gate unknown fields.

⸻

14) Open questions
	•	Can Tahoe expose APIs for separate subnets/bridges? If yes, adjust NetworkService v2 to real isolation.
	•	Best hook for in-guest file editing (hosts): via exec? cloud-init-like init? framework support?
	•	Rosetta policy toggles per-container: expose via labels or infer by arch?

⸻

15) Deliverables per phase

P0 (4–6 slices, no dates):
	•	Daemon with /var/run/docker.sock
	•	Images: pull/list
	•	Containers: create/start/stop/rm/inspect
	•	PortPublisher for -p
	•	Logs (non-TTY), Exec (non-interactive acceptable)

P1:
	•	Compose happy path (networks/volumes logical)
	•	Depends_on ordering + basic health gate
	•	Restart policies (no|on-failure|always)
	•	Interactive exec + logs -f

P2:
	•	docker build (streaming), auth, keychain
	•	Crash reconciliation, GC, events (start/stop/pull/build)

⸻

16) Acceptance tests (for your stack)
	•	docker compose up -d (FastAPI 8000, Postgres 5432, Redis 6379, worker)
	•	App resolves db:5432 and redis:6379 from within containers.
	•	curl localhost:8000/health → 200.
	•	Restart DB → app reconnects (restart policy honored).
	•	docker compose down -v cleans containers, proxies, and named volumes.

⸻

17) Minimal config
	•	YAML: none required; everything defaults sane.
	•	Env:
	•	APPLE_DOCKER_STATE_DIR=~/.apple-docker
	•	APPLE_DOCKER_DEBUG=0|1

⸻

18) Docs to ship with MVP
	•	“What works/What differs” page (one screen).
	•	Quickstart: point Docker CLI to our socket (export DOCKER_HOST=unix:///var/run/docker.sock if needed).
	•	Troubleshooting: ports in use, health timeouts, volume perms.

⸻

If you want, I’ll turn this into GitHub issues (labels by phase), plus a skeletal Swift module layout and the JSON schemas for the few Docker endpoints we’ll implement first.
