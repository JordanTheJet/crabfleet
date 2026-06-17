# Running crabfleet with ZeroClaw as the brain (no Cloudflare account, no codex)

## TL;DR
- **Control plane** (app + API + auth + D1 + R2 + Durable Objects) runs locally under `wrangler dev` (Miniflare) — **no Cloudflare account, no deploy**.
- **codex → zeroclaw**: the interactive command is a configurable string. The default is now `acpx zeroclaw` (`src/worker/session-create-request.ts`); you can also override per session via the API `command` field.
- **One hard Cloudflare dependency**: the `container` (Cloudflare Sandbox) execution plane can't be emulated by Miniflare. Use the built-in **`crabbox` runtime-adapter** path (loopback HTTP) instead.

## Local run (no account)
```sh
pnpm install && pnpm build
wrangler d1 migrations apply DB --local
# remove the "containers" block from wrangler.jsonc (skip the Docker image build)
# .dev.vars:
#   CRABFLEET_DEFAULT_RUNTIME=crabbox
#   CRABFLEET_INTERACTIVE_RUNTIMES=crabbox
#   CRABFLEET_DEV_LOGIN_ENABLED=true
#   CRABBOX_RUNTIME_ADAPTER_URL=http://127.0.0.1:8088
#   CRABBOX_RUNTIME_ADAPTER_TOKEN=local-dev-token
#   CRABBOX_RUNTIME_ADAPTER_NAMESPACE=local
#   CRABBOX_INTERACTIVE_PROVISION_TOKEN=local-dev-token
wrangler dev      # log in at http://localhost:8787/auth/dev , create a "crabbox" session
```

## The remaining piece: a `runtime-v1` adapter
crabfleet ships the **client** of the lifecycle adapter, not a local agent runner. Provide a small HTTP service implementing the `runtime-v1` contract (`POST /v1/workspaces` create/inspect/delete + terminal). The cleanest backend is **crabbox**: the adapter leases a box and runs `acpx zeroclaw` (or the task command) inside it — closing the loop crabfleet → crabbox → zeroclaw, entirely off Cloudflare.
