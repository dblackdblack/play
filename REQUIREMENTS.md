# Requirements — Private Deployment of Generative.fm Play

This document captures the requirements for standing up a fully private copy of the `generativefm/play` music player. It is the source of truth for the migration; the rules it imposes are codified in `CLAUDE.md` and in maintained memory.

## 1. Context

`generativefm/play` is an open-source React SPA at [play.generative.fm](https://play.generative.fm) that depends on three external services:

- `user.api.generative.fm` — account / likes / history / play-time API (**source not public**)
- `stats.api.generative.fm` — emissions + global play-time API (**source not public**)
- `samples.alexbainter.com` — public S3 bucket of mp3 sample files

The goal is a fully private deployment: frontend, backend, sample storage, and auth all under dblack's control. Both unavailable backends must be reimplemented against the client-side API contracts observed in the `@generative.fm/user` and `@generative.fm/stats` npm packages.

## 2. Scope

In scope:

- Fork of `generativefm/play` deployed as a static SPA on GCP + Cloudflare.
- Go backend reimplementing the `user` and `stats` HTTP APIs against Neon Postgres.
- Google OAuth auth restricted to a single allowlisted email.
- Full mirror of the sample mp3 library to a GCS bucket behind Cloudflare.
- Infrastructure fully described in Terraform (with specific carve-outs noted in §6).
- CI/CD that gates deploys on a green test suite and 100% coverage.

Out of scope:

- Migration of existing public-instance user data into the private instance.
- Multi-tenant / multi-user support beyond the single allowlisted email.
- Native-app build of play (upstream supports one; we drop it).
- Sentry or any external error reporting.

## 3. Architecture

```
 ┌──────────────────────────────┐       ┌─────────────────────────────┐
 │  Cloudflare zone             │       │  Cloudflare zone            │
 │  play.<domain>               │       │  api.play.<domain>          │
 │  samples.play.<domain>       │       └──────────────┬──────────────┘
 └───────────┬──────────────────┘                      │ proxy
             │ proxy                                    ▼
             ▼                           ┌──────────────────────────────┐
 ┌──────────────────────┐                │  Cloud Run (Go 1.26.2)       │
 │  GCS: SPA bucket     │                │   GET  /v1/user/:id          │
 │  GCS: samples bucket │                │   POST /v1/user/:id/actions  │
 └──────────────────────┘                │   POST /v1/emissions         │
                                          │   GET  /v1/global/playtime  │
                                          └──────────────┬───────────────┘
                                                         │ pgx (pooled)
                                                         ▼
                                          ┌──────────────────────────┐
                                          │  Neon Postgres           │
                                          │  users, emissions        │
                                          └──────────────────────────┘
```

Client auth: Google Identity Services → ID token (JWT, `aud = GOOGLE_OAUTH_CLIENT_ID`). Backend verifies via `google.golang.org/api/idtoken`, rejects any `email` other than `dblack@dblack.net`.

## 4. Stack decisions

| Layer | Choice |
|---|---|
| Frontend language | **TypeScript** (strict mode). All `.js`/`.jsx` converted during the modernization pass. |
| Frontend framework | **React 19** (`createRoot`, concurrent mode, StrictMode on in dev) |
| Frontend routing | **react-router-dom v7** (data router API) |
| Frontend state | **Redux 5 + Redux Toolkit** (`configureStore`, `createSlice`, `createAsyncThunk`) + **react-redux 9** |
| Frontend UI kit | **MUI v7** (`@mui/material`, `@mui/icons-material`) with Emotion v11 styling engine |
| Frontend build | **Vite** (dev + prod build). Replaces webpack. Custom `.gfm.manifest.json` loader → small Vite plugin. `vite-plugin-pwa` bundles Workbox + favicons + web manifest handling. |
| Frontend lint | **ESLint 9 + flat config** (`eslint.config.js`) with `typescript-eslint` v8. Replaces the classic `.eslintrc`. |
| Frontend service worker | **Workbox** (via `vite-plugin-pwa`). Replaces the hand-rolled `src/service-worker/sw.js`. Precaches the SPA shell + fonts; runtime `StaleWhileRevalidate` for sample mp3s. Handles cache versioning via build-time manifest injection. |
| Frontend test runner | **Vitest + React Testing Library + jsdom** (replaces Karma+Mocha) |
| Frontend e2e | Cypress (existing, kept) |
| Auth provider | Google Identity Services (Google OAuth web client), email allowlist `dblack@dblack.net` |
| Backend language / runtime | **Go 1.26.2** on Cloud Run |
| Backend HTTP router | **Go stdlib `net/http.ServeMux`** (Go 1.22+ method-pattern routing). No chi dep. |
| Backend DB driver (prod) | `github.com/jackc/pgx/v5` with `pgxpool` (against Neon) |
| Backend DB driver (test) | `modernc.org/sqlite` — pure-Go, no CGO, in-memory `:memory:` for unit + integration tests |
| Backend DB access layer | **`sqlc`** — generates type-safe Go from SQL; emits both `internal/db/pg/` and `internal/db/sqlite/` from the same `.sql` sources |
| Backend auth lib | `google.golang.org/api/idtoken` |
| Backend migrations | `github.com/golang-migrate/migrate/v4` (or `tern` — pick at implementation) |
| Backend repo layout | `play-api/` subdirectory of this repo |
| Database | **Neon Postgres** (pooled connection; managed manually via Neon console / `neonctl`) |
| Static hosting | **GCS bucket behind Cloudflare** (uniform bucket-level access, public read; Cloudflare proxies + caches) |
| Samples hosting | Separate **GCS bucket** mirrored from public `samples.alexbainter.com` |
| CDN / DNS | **Cloudflare** (existing zone) |
| Container registry | Artifact Registry in the workload project |
| Error reporting | **Dropped** (remove all Sentry integrations) |
| Secrets store | **Google Secret Manager** |
| IaC | **Terraform** (managed via **tfenv**), state on GCS with native locking |
| Domain pattern | `play.<domain>` / `api.play.<domain>` / `samples.play.<domain>` |

## 4a. Frontend dependency modernization

Upstream `@generative.fm/play` targets React 17 + MUI v4 + router v5 + Redux 4, with helper packages (`@generative.fm/web-ui`, `/user`, `/stats`) that peer-pin those versions. We modernize wholesale.

**Strategy: inline the upstream helper packages and modernize in place.**

The three MIT-licensed helper packages are each small enough to copy into this repo. No cross-repo coordination, no npm overrides, no peer-dep gymnastics.

| Upstream package | New home in this repo | Notes |
|---|---|---|
| `@generative.fm/web-ui` | `src/ui/` | Rewrite against MUI v7 + Emotion. Drop JSS/`makeStyles` in favor of `styled()` + `sx`. |
| `@generative.fm/user` | `src/user-sync/` | Rewrite the middleware as a Redux Toolkit slice + `createAsyncThunk`. Same wire format to the backend. |
| `@generative.fm/stats` | `src/stats-sync/` | Same pattern as user-sync. |

The `@alexbainter/indexed-db` helper is replaced by `idb` (Jake Archibald) — smaller, actively maintained, first-class TypeScript types.

**Version targets:**

| Dep | From → To |
|---|---|
| `react` / `react-dom` | 17.0.x → **19.x** |
| `react-redux` | 7.2.x → **9.x** |
| `redux` | 4.0.x → **5.x** + `@reduxjs/toolkit` |
| `react-router-dom` | 5.2.x → **7.x** |
| `@material-ui/*` v4 | → **`@mui/material` + `@mui/icons-material` v7** + `@emotion/react` + `@emotion/styled` |
| `tone` | 14.7.x → **15.x** (with fallback — see below) |
| `@alexbainter/indexed-db` | → `idb` |
| `@auth0/auth0-react` | **removed** |
| `@sentry/*` | **removed** |
| `prop-types` | **removed** (TypeScript types replace it) |
| `cypress` | 6.5.x → **14.x** |
| `webpack` asset loaders | `file-loader` + `image-webpack-loader` → webpack 5 **asset modules** |
| Babel presets | → latest patch of `@babel/preset-env`, `@babel/preset-react`, `@babel/preset-typescript` |
| Node toolchain | → **22 LTS** (pinned in `.nvmrc`) |

**Tone.js upgrade:**
Attempt v15 optimistically. The `@generative-music/pieces-alex-bainter` package peer-pins `tone ^14.7.58` — a soft constraint once we control the pieces-loading code. Plan:
1. Upgrade to Tone v15.
2. Write a smoke test (`src/pieces/__tests__/piece-smoke.test.ts`) that imports each piece manifest, schedules a short (~3s) playback against an `OfflineAudioContext`, and asserts no errors and non-silent output.
3. If any piece breaks, pin back to Tone 14.x and revisit later.

**TypeScript migration:**
- `tsconfig.json` with `strict: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`.
- Done in topic branches feeding into `private-deploy`: (1) store/middleware/TS-infra scaffolding, (2) inlined helper packages, (3) components by feature directory.
- Incremental with `allowJs: true` during the migration; flip to `allowJs: false` once every file is `.ts`/`.tsx`.
- 100% coverage rule applies equally to TS; Vitest thresholds set on the emitted + source files.

**Biggest risks, pre-noted:**
- MUI v4 → v7 is the deepest lift. Official codemods cover ~60–70%; the rest is hand-porting `makeStyles` call sites.
- Router v5 → v7 touches many files (`useHistory` → `useNavigate`, `<Switch>` → `<Routes>`, `<Redirect>` → `<Navigate>`).
- React 19 StrictMode reveals latent double-invocation bugs in `useEffect`; fix properly rather than working around.

## 5. Test coverage and deploy gating

- **100% line, branch, function, and statement coverage** on both the frontend and the Go backend.
- CI fails the build if coverage falls below 100% on either side.
- CI fails the build if any test fails or if `go vet` / `gofmt` / ESLint report issues.
- **No deploy runs unless the full test suite is green.** The `make deploy*` targets and CI workflows both enforce this — deploy jobs depend on test + coverage jobs and cannot run if any upstream fails.
- Cypress e2e is run in CI but is defense-in-depth — not counted toward the 100% unit coverage gate.
- Allowable coverage exclusions:
  - Service-worker shims (`src/service-worker/sw.js`, `register.js`) — testable helpers extracted; thin shims excluded.
  - Generated files (favicons, build output).
  - Deleted modules (e.g., `src/sentry/*` after removal).
- Go coverage measured with `go test ./... -race -coverpkg=./... -coverprofile=cover.out`, then `go tool cover -func=cover.out` gated by a tiny script that fails if the total is not exactly `100.0%`. `main.go` is reduced to a shim so real logic lives in `internal/app.Run(...)` and is fully testable.

## 6. Infrastructure-as-code rule

**Absolute rule:** every mutation of GCP or Cloudflare state goes through Terraform. No `gcloud`/`gsutil`/`bq`/`wrangler` scripts that create, update, or delete cloud resources. Read-only calls (`gcloud ... list/describe`, `gcloud secrets versions access`) are fine.

Terraform setup:
- Version pinned in `infra/.terraform-version`, installed via `tfenv`.
- State on GCS with native object-generation-based locking (no DynamoDB-equivalent needed).
- State bucket created by Terraform itself via an initial local-state apply, then migrated to the GCS backend with `terraform init -migrate-state`.
- Providers: `hashicorp/google` ~> 6.x, `hashicorp/google-beta` ~> 6.x, `cloudflare/cloudflare` ~> 5.x.

**State hygiene carve-outs (managed outside Terraform):**

Any resource whose normal lifecycle writes a secret value into state is managed by hand via the vendor CLI/console; Terraform only references it by name string.

- **Google Secret Manager secrets + versions** — via `gcloud secrets`. Forbidden Terraform resources: `google_secret_manager_secret`, `google_secret_manager_secret_version`. Permitted references: `google_secret_manager_secret_iam_member` (by name string), Cloud Run `--set-secrets` wiring (by name string).
- **Neon** — via Neon web console or `neonctl`. The `kislerdm/neon` provider persists generated passwords in state; do not use it. Terraform never talks to Neon. The pooled connection string is stored as GSM secret `DATABASE_URL`; Cloud Run reads it through the GSM reference above.
- **GitHub repo settings** (branch protection, required checks) — configured by hand in the GitHub UI. Do not introduce the `integrations/github` Terraform provider.
- **Google OAuth client creation** — GCP Console → APIs & Services → Credentials → OAuth client. The Google Cloud API for OAuth clients is not fully public; Terraform cannot manage these cleanly. The resulting client ID is stored in GSM as `GOOGLE_OAUTH_CLIENT_ID`.

## 7. Secrets

All secrets live in **Google Secret Manager** in the workload project. Managed manually via `gcloud`:

| Secret | Purpose |
|---|---|
| `DATABASE_URL` | Neon pooled connection string |
| `GOOGLE_OAUTH_CLIENT_ID` | Google OAuth web client ID (referenced by frontend + backend) |
| `CLOUDFLARE_API_TOKEN` | Cloudflare provider auth for Terraform, and cache-purge calls from CI |
| `BACKEND_TEST_AUTH` | Shared secret enabling Claude-driven direct API testing via `X-Test-Auth` header (see §18) |

Adding a secret (one-time per secret):

```
gcloud secrets create <NAME> --project="$PROJECT_ID" --replication-policy=automatic
printf '%s' '<value>' | gcloud secrets versions add <NAME> --project="$PROJECT_ID" --data-file=-
```

Rotation: add a new version with the same `secrets versions add` command; Cloud Run revisions picking `latest` get the new value on next start.

Consumption:

- **Cloud Run** reads `DATABASE_URL` and `GOOGLE_OAUTH_CLIENT_ID` via `--set-secrets` wired in Terraform (by string name; values never enter state).
- **Terraform** reads `CLOUDFLARE_API_TOKEN` from an env var that the Makefile populates via `gcloud secrets versions access latest --secret=CLOUDFLARE_API_TOKEN`.

`.env` files may exist locally for developer convenience but must never contain real credentials.

## 7a. Pre-commit hooks

Managed via **lefthook** (`lefthook.yml` at repo root). Fast, Go-based, no Node dependency at hook time. Hooks are installed on `git clone` via `make setup` / `lefthook install`.

**`pre-commit` (must be fast — target ≤ 2s for typical diffs):**
- ESLint on staged `*.js` / `*.jsx` (`--max-warnings 0`, fix-in-place off).
- Prettier check on staged `*.{js,jsx,json,md,scss}` (uses existing `.prettierrc`).
- `gofmt -l` + `go vet` on staged `*.go`; non-zero if any file needs formatting.
- `terraform fmt -check -diff` on staged `*.tf` / `*.tfvars`.

**`pre-push` (slower checks; skippable with `--no-verify` only in rare documented cases):**
- `go test ./... -short -race` in `play-api/`.
- `npm test -- --run` on the frontend (Vitest; full unit run is fast — low seconds).
- `terraform validate` in `infra/` if any `*.tf` files changed in the push.

**`commit-msg`:**
- Optional; skipping for now. Solo project; no need for commit-message linting.

Hooks operate on staged files only (`{staged_files}` glob in lefthook). They mirror — but don't replace — CI gates: CI runs the full suite and the coverage check; hooks just catch cheap things locally. A failing hook is never the *only* thing preventing a bad commit — CI is the authoritative gate.

## 8. Git workflow

- All work happens on feature branches; the long-lived umbrella branch for this migration is `private-deploy`.
- Short-lived topic branches for individual milestones feed into `private-deploy` via PR.
- Final PR merges `private-deploy` → `main`.
- `main` is never committed to directly.
- GitHub branch protection (configured manually) requires CI green before merge.

## 9. Makefile

Single `Makefile` at repo root is the unified entry point for both local and CI.

Required targets:

```
make deploy             # full deploy (both halves, tests first)
make deploy-frontend    # frontend only
make deploy-backend     # backend only

make test               # run all tests
make test-frontend
make test-backend

make coverage           # run + enforce 100% coverage
make coverage-frontend
make coverage-backend

make lint               # eslint + go vet + gofmt
make build              # build both
make build-frontend
make build-backend

make tf-init            # tfenv use + terraform init
make tf-plan
make tf-apply

make dev-frontend       # webpack dev server
make dev-backend        # go run ./cmd/server

make samples-mirror     # populate samples bucket from public source
make vuln               # go vulnerability check (govulncheck ./...) in play-api/
make clean

make setup              # one-time: install lefthook hooks, tfenv version, npm deps, go mod download
make mirror-fonts       # one-shot: download Google Fonts, upload to SPA bucket, rewrite CSS
```

Deploy targets depend on their corresponding test + coverage targets; a deploy cannot run with a broken suite.

CI invokes the same `make` targets so local and CI paths are identical.

## 10. CI/CD

GitHub Actions workflow with jobs forming a DAG; `deploy-*` jobs depend on `lint`, `test-*`, `coverage-*`, `vuln-be`, `e2e-cypress`. Failure in any upstream job blocks the deploy.

`vuln-be` runs `govulncheck ./...` inside `play-api/`. A new HIGH-severity vuln in a transitive dep will fail the build; upgrade the dep, or add a documented `govulncheck:ignore` with an expiration date if the vuln is not exploitable in our usage.

CI authenticates to GCP via **Workload Identity Federation** (no service account JSON keys). The WIF pool + provider + binding are managed in Terraform; the GitHub repo is trusted to the SA that has `storage.objectAdmin` on the SPA bucket, `artifactregistry.writer` on the AR repo, `run.developer` on the Cloud Run service, and `secretmanager.secretAccessor` on any secrets Terraform needs to read.

Deploy sequence for the backend: build + push image with tag `sha-<shortsha>` → write that image reference into `infra/image.auto.tfvars` → `terraform apply` in `infra/` rolls Cloud Run to the new revision.

Deploy sequence for the frontend:
1. Build `dist/`.
2. `gsutil -m rsync -r -d dist/ gs://$SPA_BUCKET` with default header `Cache-Control: public, max-age=31536000, immutable` (safe because webpack contenthashes all asset filenames).
3. Overwrite `index.html` and `sw.js` with `gsutil cp -h "Cache-Control:no-cache, max-age=0, must-revalidate"` — both have stable URLs and must reflect the latest build on every visit.
4. Purge Cloudflare cache for `/` and `/sw.js` via the Cloudflare API.

Belt-and-suspenders: Terraform provisions a Cloudflare Cache Rule that forces `Cache-Control: no-cache, max-age=0, must-revalidate` on `/index.html`, `/`, and `/sw.js` regardless of origin headers, so a misconfigured gsutil call can't strand a stale shell in caches.

**Cloudflare protocol settings (verified in Terraform):** HTTP/3 (with QUIC) enabled; TLS 1.3 enabled; Automatic HTTPS Rewrites on; Brotli on. These are zone-level toggles expressible as `cloudflare_zone_setting` resources.

## 11. Client-side code changes

Against the upstream fork:

- Parameterize the Auth0 config hardcoded in `src/app/app.jsx:66-72`; replace `Auth0Provider` with a thin Google Identity Services provider that exposes the same user/token selectors the existing `@generative.fm/user` middleware consumes.
- Update the `EnvironmentPlugin` defaults in `webpack/webpack.production.config.js` to be fully overridable via env: `SAMPLE_FILE_HOST`, `GFM_STATS_ENDPOINT`, `GFM_USER_ENDPOINT`, `GOOGLE_OAUTH_CLIENT_ID`, `APP_VERSION`.
- Remove Sentry: drop `@sentry/react`, `@sentry/tracing`, `@sentry/webpack-plugin`; delete `src/sentry/`; remove the Sentry plugin block in `webpack/webpack.production.config.js:33-50`.
- Remove `@alexbainter/s3-sync` and the AWS deploy script in `package.json:85`. Replace with a GCS upload invoked from the Makefile.
- Rewrite `.github/workflows/deploy-production.yml` to drop the native build, drop AWS creds, authenticate to GCP via WIF, build + upload via `make deploy-frontend`.
- Audit `src/user/select-*.js`, `auth-button.jsx`, `user-context-menu.jsx` — anything tied to Auth0's specific shape gets re-pointed at the Google auth slice. These files become `.ts`/`.tsx` as part of the migration.
- Drop `prop-types` imports; replace with TypeScript prop types.
- All new / rewritten files are TypeScript; JS is the exception, tracked down to zero by the end of milestone 2d.

## 12. Backend (`play-api/`)

Layout:

```
play-api/
  cmd/server/main.go           # thin shim -> app.Run
  internal/app/app.go          # Run(ctx, args, stdin, stdout, stderr, env); sets up mux + middleware
  internal/auth/google.go      # ID-token verify + email allowlist
  internal/auth/testauth.go    # X-Test-Auth bypass middleware (constant-time compare)
  internal/db/DB.go            # DB interface implemented by both engines
  internal/db/pg/              # generated by sqlc (postgresql engine)
  internal/db/sqlite/          # generated by sqlc (sqlite engine)
  internal/db/queries/         # hand-written *.sql source for sqlc
  internal/db/migrations/      # *.sql (dialect-agnostic subset)
  internal/handlers/user.go
  internal/handlers/emissions.go
  internal/handlers/playtime.go
  internal/actions/reducer.go  # Go port of @generative.fm/user reducer logic
  sqlc.yaml                    # sqlc config: two output packages from the same queries
  Dockerfile
  go.mod                       # go 1.26.2
```

HTTP routing uses stdlib `net/http.ServeMux` method-pattern syntax (Go 1.22+):

```go
mux := http.NewServeMux()
mux.Handle("GET  /v1/user/{id}",         authMW(userGet))
mux.Handle("POST /v1/user/{id}/actions", authMW(userActions))
mux.Handle("POST /v1/emissions",         authOptMW(emissionsPost))
mux.Handle("GET  /v1/global/playtime",                globalPlaytime)
```

Middleware chaining is a five-line helper (no `chi` dependency).

Routes (v1):

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET  | `/v1/user/:id`           | Bearer | Return the user record, or `{user: null}` |
| POST | `/v1/user/:id/actions`   | Bearer | Apply a batch of user actions (likes/dislikes/history/play-time), return updated user |
| POST | `/v1/emissions`          | Optional Bearer | Record anonymous or authenticated playback emissions |
| GET  | `/v1/global/playtime`    | None   | Return `{ [pieceId]: totalMs }`; cached server-side; `Cache-Control: public, max-age=3600` |

Auth middleware order:
1. If `X-Test-Auth` header is present, constant-time-compare against `BACKEND_TEST_AUTH` (from env; loaded from GSM). Match → synthetic auth context (see §18). No match → 401 immediately, do **not** fall through to the Google path.
2. Otherwise, `OAuth2Client.verifyIdToken({ idToken, audience: GOOGLE_OAUTH_CLIENT_ID })`; reject if `payload.email != "dblack@dblack.net"`; set `req.userId = payload.sub`; enforce `:id` param matches `req.userId`.

All request/response logging redacts the `Authorization` and `X-Test-Auth` headers; request bodies on user-data routes are logged with `email` and any other identifying fields redacted.

Database schema — written in a **dialect-agnostic subset** so the same migrations work in Postgres (prod) and SQLite (tests). No `jsonb`, no `timestamptz`, no Postgres-specific JSON operators. All JSON manipulation happens in Go (`json.Unmarshal` / `json.Marshal`) against `text` columns guarded by `json_valid` checks. All timestamps are `bigint` epoch milliseconds.

```sql
create table users (
  id              text primary key,
  email           text not null,
  likes           text not null default '[]'     check (json_valid(likes)),
  dislikes        text not null default '[]'     check (json_valid(dislikes)),
  history         text not null default '[]'     check (json_valid(history)),
  play_time       text not null default '{}'     check (json_valid(play_time)),
  anonymous_data  text not null default '{}'     check (json_valid(anonymous_data)),
  updated_at_ms   bigint not null
);

create table emissions (
  id            text primary key,
  user_id       text references users(id),
  piece_id      text not null,
  start_time_ms bigint not null,
  end_time_ms   bigint not null,
  created_at_ms bigint not null
);

create index emissions_piece_id_idx on emissions (piece_id);
```

Trade-off accepted: we forgo Postgres features like GIN indexes on JSON, `LISTEN/NOTIFY`, and JSON path operators. None are needed by the current API. Revisiting this is a one-migration change if ever required.

Container: `FROM golang:1.26.2-alpine AS build` → `gcr.io/distroless/static-debian12`. Env: `PORT`, `DATABASE_URL`, `GOOGLE_OAUTH_CLIENT_ID`, `ALLOWED_EMAILS` (default `dblack@dblack.net`).

Cloud Run: `allow-unauthenticated` at the Cloud Run level (we gate at Cloudflare + app-level JWT verification). Cost-conscious configuration:

- `min-instances = 0` (scales to zero when idle)
- `max-instances = 3` (hard cap on runaway cost)
- `concurrency = 80` (default; plenty of headroom per instance)
- CPU allocated **only during request processing** (not always-on)
- **Request-based billing** (cheaper than instance-based for low-traffic apps)
- Memory 512 MiB (default; grow only if profiling says so)

## 13a. Font mirror

To avoid leaking the user's IP to Google on every page load, Google Fonts are self-hosted from the SPA bucket. One-shot Go command in `play-api/cmd/mirror-fonts/main.go`:

1. Fetch `https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap`.
2. Parse out every `url(https://fonts.gstatic.com/...)` reference; download each woff2/woff/ttf; upload to `gs://<spa-bucket>/fonts/gstatic/<hashed-path>` with `Cache-Control: public, max-age=31536000, immutable`.
3. Rewrite the CSS to reference relative `/fonts/gstatic/...` URLs; upload as `gs://<spa-bucket>/fonts/roboto.css`.
4. Commit the rewritten CSS filename/hash into the repo (as a small JSON artifact) so the frontend build doesn't require network access to pick it up.
5. Patch `src/service-worker/sw.js` (currently references `fonts.googleapis.com` and `fonts.gstatic.com` directly — see `src/service-worker/sw.js:2-3`) to use the self-hosted CSS + origin. Same for any `<link rel="stylesheet">` in `src/index.template.html`.

Invoked via `make mirror-fonts`. Re-run only when the upstream font set changes.

Result: zero third-party requests on page load; single origin; no IP leak to Google.

## 13b. Sample mirror

One-shot Go command in `play-api/cmd/mirror-samples/main.go` (shares the Go toolchain already set up for the backend; no separate Node script):

1. After `npm install` locally, read sample manifests from `@generative-music/samples-alex-bainter` and piece manifests from `@generative-music/pieces-alex-bainter`.
2. For each referenced URL under `https://samples.alexbainter.com/...`, download and upload to `gs://<samples-bucket>/<same-path>` with `Cache-Control: public, max-age=31536000, immutable`. Skip if already present (size + crc32c match).
3. Invoked via `make samples-mirror`. Not part of the normal deploy flow (run once, re-run only when upstream samples are updated).

Expected size: several GB; initial sync is slow but one-time. Subsequent runs are incremental.

The frontend build sets `SAMPLE_FILE_HOST=https://samples.play.<domain>`.

## 14. Milestones

| # | Milestone | Notes |
|---|---|---|
| 0 | Feature branch + Makefile skeleton + `.env.example` | set up the repo scaffolding |
| 1 | `infra/` bootstrap: tfenv + provider pinning + local-state apply + migrate-state to GCS backend | Terraform manages the project, API enablement, and state bucket |
| 2a0 | Frontend: webpack→Vite migration + ESLint 9 flat config; Karma→Vitest migration; hit 100% on the existing (React 17) code first | establishes modern tooling + coverage gate before the rewrite. Includes porting the `.gfm.manifest.json` loader to a Vite plugin. |
| 2b | Frontend: TypeScript scaffolding (`tsconfig.json`, `.ts`/`.tsx` rename of store/middleware/entry point, `allowJs: true`) | lets subsequent milestones ship TS code without a big-bang flip |
| 2c | Frontend: inline `@generative.fm/{web-ui,user,stats}` into `src/ui`/`src/user-sync`/`src/stats-sync`; drop the npm deps | prerequisite to MUI + Redux upgrades |
| 2d | Frontend: dep modernization pass (React 19 + router v7 + Redux 5 + RTK + MUI v7 + Emotion; Workbox via vite-plugin-pwa replaces the hand-rolled service worker) | staged per-topic-branch; each landing green at 100% coverage |
| 2e | Frontend: Tone.js v15 attempt with per-piece smoke test; fall back to 14.x if needed | — |
| 3 | Backend: Go skeleton + test harness + 100% coverage gate from day 1 | — |
| 4 | Backend: sqlc + SQLite in-memory test harness + endpoints + migrations; Neon integration smoke in CI | unit tests run in ms against SQLite; a single CI job runs the same tests against a real Neon branch as smoke |
| 5 | TF: Artifact Registry + Cloud Run service + WIF pool + IAM | Neon and GSM secrets exist manually before this step |
| 6 | Frontend: replace Auth0 with Google Identity Services; drop Sentry; parameterize endpoints | — |
| 7 | TF: GCS buckets + Cloudflare DNS + cache rules + SPA history fallback (Worker or Transform Rule) + Cloud Run domain mapping | — |
| 8 | Sample mirror command + populate samples bucket | — |
| 9 | CI workflow: gated deploys, WIF, TF apply | — |
| 10 | End-to-end shake-out | real playback, likes, history, cross-device sync |

## 15. Implementation prerequisites (to provide at kickoff)

Not needed to finalize the plan; needed the moment we start implementing:

1. **GCP project ID** (globally unique; e.g. `play-private-dblack`).
2. **GCP billing account ID** (`gcloud billing accounts list`).
3. **GCP org / folder ID**, if any (otherwise no-org project).
4. **Parent domain name** and **Cloudflare zone ID + account ID**.
5. **Cloudflare API token** scoped to the zone with: `Zone.Zone:Read`, `Zone.DNS:Edit`, `Zone.Cache Purge:Purge`, `Zone.Cache Rules:Edit`, `Zone.Page Rules:Edit` (and `Workers Routes:Edit` if we use a Worker for SPA history fallback). Created manually; stashed in GSM as `CLOUDFLARE_API_TOKEN`.
6. **Google OAuth client ID** (Web application, origin `https://play.<domain>`, redirect URI also the origin). Created manually in the GCP console; stashed in GSM as `GOOGLE_OAUTH_CLIENT_ID`.
7. **Neon project + role + pooled connection string**. Created manually via the Neon console or `neonctl`; connection string stashed in GSM as `DATABASE_URL`.

## 15a. Cost posture

Target: **single-digit dollars per month** for a lightly used single-user deployment.

- GCP billing budget + email alert provisioned in Terraform at $10/mo with 50%/90%/100% thresholds. Cheap insurance against runaway egress or a misconfigured cache.
- Cloud Run: scale-to-zero (`min=0`), capped at `max=3`, request-based billing, CPU only during requests.
- Neon: free tier with autosuspend; first query after idle incurs ~2s wake-up. Accepted.
- GCS: STANDARD storage class for both buckets (samples are accessed often enough that NEARLINE's retrieval fees would be a net loss). Cloudflare caching in front eliminates most egress.
- Artifact Registry: lifecycle policy retains the last 10 image tags; older tags auto-delete. Keeps registry size in the pennies.
- Cloudflare: free plan covers DNS + proxy + Cache Rules + Transform Rules + HTTP/3. Prefer Transform Rules over Workers where both solve the same problem (Transform Rules are always free; Workers have a free tier but also a usage meter).
- Logging: Cloud Run logs to Cloud Logging default retention (30 days); ingest stays well under the free-tier quota.
- Secret Manager: first 6 active secret versions per secret are free; we're well under.
- No paid tier upgrades (Neon, Sentry, APMs, monitoring add-ons) unless a concrete need emerges.

## 16. Defaults (override at any time)

| Setting | Default | Change by |
|---|---|---|
| Long-lived branch name | `private-deploy` | tell me otherwise |
| Terraform version | latest stable 1.x at implementation time | pin your own in `infra/.terraform-version` |
| Cloud Run region | `us-central1` | set `region` var in `infra/terraform.tfvars` |
| Neon region | `us-east-2` (AWS) | pick any region in the Neon console |
| GCP organization | none (standalone project) | provide `org_id` or `folder_id` |
| Google OAuth consent screen mode | External → Testing, sole test user `dblack@dblack.net` | no change expected; this keeps us out of Google's verification flow |
| Neon backup retention | Neon default (7-day PITR on free tier) | upgrade tier for longer retention |
| Budget | under $10/mo, dominated by samples egress if used heavily | — |

## 18. Claude-driven direct API testing (test-auth bypass)

The backend exposes an auth bypass for Claude to hit the API directly (e.g., smoke tests, data inspection, regression repro) without going through an interactive Google OAuth flow.

**Mechanism:**
- Header: `X-Test-Auth: <secret>`.
- Secret stored in GSM as `BACKEND_TEST_AUTH` (32 random bytes, base64url).
- Cloud Run reads it via `--set-secrets=BACKEND_TEST_AUTH=BACKEND_TEST_AUTH:latest` (string reference, not in Terraform state).
- Backend middleware uses `subtle.ConstantTimeCompare`. Valid match sets `req.userId = "dblack@dblack.net"` (or `"test:dblack@dblack.net"` — TBD: does dblack prefer the bypass to impersonate the real user or act as a separate test identity?).
- Present-but-invalid → 401 immediately (no fallthrough to the Google path).

**Operational rules for Claude (enforced in `CLAUDE.md` + memory):**
- **Never** run `gcloud secrets versions access --secret=BACKEND_TEST_AUTH` as a standalone command — its stdout is the secret value and will land in the tool result.
- **Always** inline it with command substitution directly into `curl`:
  ```
  curl -sS -H "X-Test-Auth: $(gcloud secrets versions access latest \
       --secret=BACKEND_TEST_AUTH --project=\"$PROJECT_ID\" --quiet)" \
       https://api.play.<domain>/v1/...
  ```
  The Bash tool sees the literal `$(...)`; the expanded value only exists in the curl process's argv, never in tool output.
- **Never** use `curl -v`, `--trace`, or `--trace-ascii` with this secret — they echo headers.
- **Never** `echo`, `cat`, `tee`, or assign the secret to a shell variable that might be printed later in the same Bash call.
- **Never** write the secret to a file.

**Backend-side secret handling:**
- Constant-time compare against the env-loaded value.
- Logger middleware redacts `Authorization` and `X-Test-Auth` from any log line.
- No request body logging on authenticated routes beyond what's strictly required for debugging.

**Rotation runbook:**
```
openssl rand -base64 32 | tr -d '\n' \
  | gcloud secrets versions add BACKEND_TEST_AUTH \
      --project="$PROJECT_ID" --data-file=-
gcloud run services update play-api --region=us-central1 \
  --project="$PROJECT_ID"   # triggers new revision that picks up :latest
```

**Risk acknowledgement:** this is a backdoor by design. Anything that knows the secret has full API access as dblack. Mitigations: GSM-only storage, not in repo/logs/TF state; constant-time compare; trivially rotatable. If at any point the value is suspected to have leaked (e.g., it surfaced in a tool result), rotate immediately.

A stronger alternative is available (Google SA ID tokens with audience-bound JWTs); not adopted per explicit request for a shared-secret approach.

## 19. Non-goals (explicit)

- Not re-hosting the upstream public instance's data or proxying it.
- Not supporting multiple users.
- Not shipping a native app (Electron / Tauri) build.
- Not running a separate staging environment (single production-only deployment; PR previews out of scope for now).
- Not integrating external monitoring / APM.
- The `X-Test-Auth` bypass is for Claude-driven API testing only — not a general-purpose API key for external clients.
- **Cold-start latency on first use is accepted.** Cloud Run stays at `min-instances = 0`; Neon stays on the autosuspend plan. First hit of the day incurs ~2–4s of wake-up delay across both services. No keep-warm pings, no paid-tier Neon upgrade, no `select 1` priming in uptime checks. Cost wins over snappiness here.
