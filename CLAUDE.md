# CLAUDE.md

Guidance for Claude Code working in this repository. A sibling `AGENTS.md` covers
the same ground for other agents; this file is tailored for Claude Code. If a fact
here ever conflicts with a source/config file, trust the source file.

## What this project is

**Memos** (`github.com/usememos/memos`) is an open-source, self-hosted, Markdown-native
note-taking app. It ships as a single Go binary that serves a JSON/gRPC API and the
built React SPA. Storage is pluggable across SQLite, MySQL, and PostgreSQL.

- **Backend:** Go 1.26.2, Echo v5, Connect RPC + gRPC-Gateway, Protocol Buffers.
- **Frontend:** React 19, TypeScript 6, Vite 8, Tailwind CSS v4, React Query v5, Biome.
- **Generated API:** `proto/gen/` (Go + OpenAPI), `web/src/types/proto/` (TypeScript).

## Layout

| Path | Purpose |
| --- | --- |
| `cmd/memos/main.go` | Cobra/Viper CLI and server startup |
| `server/server.go` | Echo HTTP server and background runner wiring |
| `server/auth/` | JWT access/refresh tokens, PAT handling |
| `server/router/api/v1/` | Connect/gRPC-Gateway services, ACL config, SSE hub |
| `server/router/frontend/` | Static SPA serving (release build lands in `dist/`) |
| `server/router/fileserver/` | File serving, thumbnails, range requests |
| `server/router/{mcp,rss}/` | MCP server and RSS/Atom feeds |
| `server/runner/` | Background memo processing, S3 presign refresh |
| `store/` | Store facade, cache, `driver.go` interface, `migrator.go` |
| `store/db/{sqlite,mysql,postgres}/` | Per-driver implementations and SQL |
| `store/migration/{sqlite,mysql,postgres}/` | Migrations + `LATEST.sql` per driver |
| `proto/api/v1/` | Public API service definitions |
| `proto/store/` | Internal storage proto messages |
| `internal/` | App-private packages: `ai`, `cron`, `email`, `filter` (CEL), `markdown`, `idp`, `storage` (S3), `scheduler`, `webhook`, etc. |
| `web/src/connect.ts` | Connect RPC clients, auth interceptor, token refresh |
| `web/src/hooks/` | React Query hooks for server state |
| `web/src/contexts/` | React context for client/UI state |
| `web/src/components/` | Radix/Tailwind UI + feature components |

## Commands

Run from the repo root unless a command starts with `cd`.

### Backend (Go)

```bash
go run ./cmd/memos --port 8081     # Start backend dev server
go test ./...                      # All Go tests
go test -v ./store/...             # Store tests (DB drivers via Testcontainers)
go test -v -race ./server/...      # Server tests with race detector
go test -v -race ./internal/...    # Internal package tests with race detector
go mod tidy -go=1.26.2             # Match the CI tidy check
golangci-lint run                  # Lint (config: .golangci.yaml)
golangci-lint run --fix            # Auto-fix lint (includes goimports)
```

### Frontend (`web/`, pnpm)

```bash
cd web && pnpm install             # Install deps (pnpm@11.0.1, Node >= 24)
cd web && pnpm dev                 # Vite dev server, proxies API to :8081
cd web && pnpm lint                # tsc --noEmit --skipLibCheck && biome check src
cd web && pnpm lint:fix            # biome check --write src
cd web && pnpm test                # Vitest unit tests
cd web && pnpm build               # Production build
cd web && pnpm release             # Build SPA into server/router/frontend/dist
```

### Protocol Buffers (`proto/`, buf)

```bash
cd proto && buf generate           # Regenerate Go + OpenAPI + TypeScript
cd proto && buf lint               # Lint proto files
cd proto && buf format -w          # Format proto files
```

`buf.gen.yaml` writes Go/gRPC/Connect/Gateway + OpenAPI into `proto/gen/` and
TypeScript into `web/src/types/proto/`. Never hand-edit generated outputs — edit the
`.proto` source and regenerate.

## Architecture notes

- **API:** Services are defined in `proto/api/v1/`. The server exposes them through
  Connect RPC and gRPC-Gateway in `server/router/api/v1/`. Public (unauthenticated)
  routes must be registered in `server/router/api/v1/acl_config.go`.
- **Store drivers:** `store/driver.go` defines the `Driver` interface; each backend
  in `store/db/{sqlite,mysql,postgres}/` implements it. The `store/` facade and cache
  sit on top; `migrator.go` drives schema setup.
- **Migrations:** Schema changes require migrations for **all three** drivers plus an
  update to each driver's `LATEST.sql`. Fresh-install SQL and incremental migrations
  must stay equivalent.
- **CLI/config:** Cobra/Viper in `cmd/memos`. The single binary embeds and serves the
  released SPA from `server/router/frontend/dist`.

## Conventions

### Go

- Wrap errors with `errors.Wrap(err, "context")` from `github.com/pkg/errors`; do not
  use `fmt.Errorf` for wrapping.
- Return service errors via `status.Errorf(codes.X, "message")`.
- Import groups: stdlib, third-party, then `github.com/usememos/memos`. goimports runs
  under golangci-lint.
- Doc-comment exported identifiers (godot enforces punctuation).

### Frontend

- Use `@/` for absolute imports.
- Biome formatting: 2-space indent, double quotes, semicolons, 140-char line width
  (`web/biome.json`).
- Server data goes in React Query hooks under `web/src/hooks/`; UI-only state lives in
  contexts/component state.
- Tailwind v4 utilities, `cn()` for class merge, CVA for variants; reuse existing Radix
  primitives before adding new ones.
- Treat `web/src/types/proto/` as generated — no manual edits.

## CI / lint / format

- **Backend** (`.github/workflows/backend-tests.yml`): Go 1.26.2; checks `go mod tidy
  -go=1.26.2` cleanliness, golangci-lint v2.11.3; test matrix `store` / `server` /
  `internal` / `other`. Store tests run all drivers; others run with `DRIVER=sqlite`.
- **Frontend** (`.github/workflows/frontend-tests.yml`): Node 24, pnpm 11.0.1; runs
  `pnpm lint`, `pnpm test`, `pnpm build`.
- **Proto** (`.github/workflows/proto-linter.yml`): `buf lint` + `buf format` check.
- **Release:** `release-please.yml` / `release.yml`; Docker via `scripts/Dockerfile`.

## Gotchas

- Don't hand-edit generated files (`proto/gen/`, `web/src/types/proto/`) — change the
  source and regenerate.
- A schema change is incomplete without migrations + `LATEST.sql` for all three drivers.
- New public/unauthenticated endpoints must be added to `acl_config.go`.
- Keep proto changes backward-compatible unless a breaking change is explicitly intended.
- Keep diffs scoped: avoid repo-wide cleanup, dependency churn, or generated-file
  rewrites unless the task requires them.

## Verifying changes

Run the narrowest relevant check while iterating, then the check matching the changed
surface before finishing:

| Change | Verify |
| --- | --- |
| Server / router behavior | `go test -v -race ./server/...` |
| Store / migration | `go test -v ./store/...` |
| Internal package | `go test -v -race ./internal/...` |
| Frontend | `cd web && pnpm lint && pnpm test` |
| Frontend prod output | `cd web && pnpm build` (or `pnpm release`) |
| Proto API | `cd proto && buf generate && buf lint` |
