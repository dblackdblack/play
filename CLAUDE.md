# CLAUDE.md

Rules for AI assistants working in this repo. These are hard requirements, not guidelines.

## Infrastructure-as-code

**Every** mutation of GCP or Cloudflare state goes through Terraform, with exactly **one exception** (secrets — see below).

- Do not write, propose, or execute shell scripts that call `gcloud`, `gsutil`, `bq`, `wrangler`, or similar to create, update, or delete cloud resources.
- Project creation, API enablement, billing linkage, and state bucket creation are all expressible in Terraform (`google_project`, `google_project_service`, `google_storage_bucket`, then `terraform init -migrate-state` to move from local to GCS state once the bucket exists).
- Read-only calls (`gcloud ... list`/`describe`, `gcloud secrets versions access <name>`) are fine.
- Use `tfenv` to pin the Terraform version (`.terraform-version` file is the source of truth).

## Secrets and state hygiene

Over-arching principle: any resource whose normal lifecycle writes a secret value into Terraform state is managed **outside** Terraform, by hand, via the relevant vendor CLI or console. Terraform only references such resources by name/identifier as strings.

Current members:

- **Google Secret Manager secrets and versions.** Managed via `gcloud secrets create`, `gcloud secrets versions add`, `gcloud secrets versions access`. The resources `google_secret_manager_secret` and `google_secret_manager_secret_version` are forbidden. Terraform may use `google_secret_manager_secret_iam_member` (secret referenced as a string) and Cloud Run `--set-secrets` (string reference) to wire secrets into consumers — these do not place values in state.
- **Neon (Postgres).** The `kislerdm/neon` Terraform provider generates role passwords into state, so we do not use it. Create Neon projects, branches, endpoints, roles, and databases via the Neon web console or `neonctl` CLI. Put the resulting connection string into GSM as `DATABASE_URL` by hand; Cloud Run reads it via the GSM reference above.
- **GitHub repo settings** (branch protection, required checks, etc.) are configured by hand in the GitHub UI — **not** managed through Terraform. Do not introduce the `integrations/github` provider.

All secrets live in **Google Secret Manager** in the project that hosts the workload. Never commit secrets to the repo. `.env` files may exist locally for developer convenience but must not contain real credentials.

## Git workflow

- Never commit directly to `main`.
- All work happens on a feature branch; changes land on `main` through a PR.
- This applies to infra (Terraform) changes as well.

## Testing + deploy gates

- Maintain **100%** test coverage on both the frontend and the Go backend.
- CI must fail if coverage is below 100% on either side.
- No deploy runs without a fully green test suite.
- The Makefile targets (`make deploy`, `make deploy-frontend`, `make deploy-backend`) enforce the same contract locally as CI enforces remotely.

## Backend test-auth bypass

The backend accepts an `X-Test-Auth: <secret>` header as an alternative to the Google OAuth flow, used by Claude for direct API testing. The secret lives in Google Secret Manager as `BACKEND_TEST_AUTH`.

When using this secret from a Bash tool call, the value must never appear in tool stdout/stderr (which is captured and sent upstream). Required pattern:

```
curl -sS -H "X-Test-Auth: $(gcloud secrets versions access latest \
      --secret=BACKEND_TEST_AUTH --project=\"$PROJECT_ID\" --quiet)" \
      https://api.play.<domain>/v1/...
```

Never:
- Run `gcloud secrets versions access --secret=BACKEND_TEST_AUTH` as a standalone command (its stdout is the secret).
- `echo`, `cat`, `printf`, `tee` the secret or a file containing it.
- Assign the secret to a shell variable you might later print in the same Bash call.
- Invoke `curl -v`, `--trace`, or `--trace-ascii` with this secret — they echo request headers.

Backend requirements for this path:
- Constant-time compare (`subtle.ConstantTimeCompare`).
- Middleware redacts `Authorization` and `X-Test-Auth` from all log output.
- Valid → synthetic auth context (`userId = "dblack@dblack.net"` or `"test:dblack@dblack.net"` — TBD); invalid → 401 with no fallthrough to the Google path.
- Secret wired into Cloud Run via `--set-secrets=BACKEND_TEST_AUTH=BACKEND_TEST_AUTH:latest` (name reference, no value in TF state).

If the secret value ever shows up in a tool result, stop, notify dblack, and rotate immediately:
```
openssl rand -base64 32 | tr -d '\n' | gcloud secrets versions add BACKEND_TEST_AUTH \
    --project="$PROJECT_ID" --data-file=-
```
Then redeploy Cloud Run to pick up the new version.

## Planning vs implementation

- When the user says we are in planning mode, stay in planning mode until the user explicitly switches to implementation.
- In planning mode: no writes to repo files, no running of mutating commands, no pre-staging of artifacts.
- Plan deliverables live in the conversation and in `~/.claude` memory, not in the working tree.
