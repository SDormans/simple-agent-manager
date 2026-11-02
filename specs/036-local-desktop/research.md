# Research: Native Local SAM Desktop

**Date**: 2026-11-02
**Scope**: Architecture research for the clarified Windows-first, single-user local SAM feature. No application code was changed.

## Decisions and Findings

### 1. Native shell and service lifecycle

**Decision**: Recommend a packaged Electron shell with the existing React/Vite UI; this is a proposal awaiting user approval.

**Evidence and rationale**:

- The repository has a React/Vite control plane in `apps/web/`, but no Electron, Tauri, or other desktop shell package.
- Electron can reuse the existing frontend and supervise local child processes in its main process. It also exposes `safeStorage`, which uses Windows DPAPI for per-user protection, but its documented Windows model does not protect secrets from other applications running as the same user.
- Tauri is a valid alternative: it uses the OS WebView and limits frontend access through explicit IPC capabilities, but introduces Rust into the desktop supervisor and a different process-management boundary.
- A browser-only/Vite app cannot provide the requested native install and controlled stop/background behavior.

**Alternatives considered**:

| Option | Benefits | Costs / risks | Assessment |
|---|---|---|---|
| Electron | Reuses React and TypeScript; main process can own lifecycle and Windows-protected storage. | Larger bundle; renderer must remain unprivileged; same-user malware can use the user context. | Recommended for implementation effort, pending approval. |
| Tauri | Smaller shell; explicit Rust-to-webview capability boundary; system WebView. | Adds Rust and sidecar/IPC integration work; still needs secure local API and workspace services. | Strong alternative if the owner prioritizes shell isolation/size. |
| Browser + local dev servers | Reuses current development workflow. | Not a packaged native app and current dev mode does not support real agent sessions or VM/workspace provisioning. | Rejected. |

### 2. Local control plane and persistent storage

**Decision**: First validate a pinned Workerd/Miniflare runtime with the existing Worker, D1, KV, R2, and Durable Object APIs. Do not yet treat Wrangler's development command as a shippable desktop runtime.

**Evidence and rationale**:

- `apps/api/src/index.ts` exports the Hono Worker; `apps/api/package.json` uses `wrangler dev` for local development. `@hono/node-server` is declared but not used to start the API as a Node server.
- `apps/api/src/env.ts` requires Cloudflare-specific bindings including D1, KV, R2, Workers AI, and many Durable Object namespaces. `apps/api/wrangler.toml` also defines DO migrations, bindings, and scheduled triggers.
- `ProjectData` stores project chat/session data in embedded SQLite and provides real-time delivery. Replacing it with a new SQLite service would require a broad migration of storage, alarms, RPC, and WebSocket semantics.
- Cloudflare's local-development documentation says Wrangler and the Vite plugin run Worker code locally using Miniflare/Workerd and simulate bindings. Local resources persist under `.wrangler/state` by default, and local data is separate from production data. The docs describe this as local development, not as a packaged product runtime.
- Cloudflare explicitly warns that Miniflare/Workerd local execution is not a hardened sandbox for untrusted Worker code. The proposed desktop would run only trusted SAM API code there; repository/agent code remains in Docker workspaces and must not be evaluated by Workerd.
- Workers AI bindings still run models remotely, so local-mode code must not rely on `AI` for local inference. Scheduled jobs and Cloudflare account operations must be disabled or rejected in local mode rather than provided fake credentials.

**Alternatives considered**:

| Option | Benefits | Costs / risks | Assessment |
|---|---|---|---|
| Local Worker runtime with simulated bindings | Reuses Hono routes, D1 migrations, DO RPC/storage, and much of chat persistence. | Product packaging/support is unproven; required bindings and cloud-only routes need explicit local-mode handling. | Best first feasibility spike; release-blocking validation. |
| Node Hono server + SQLite | Uses a standard local server process and can reuse Hono route composition. | The API is not currently Node-startable; D1 and DO contracts, alarms, hibernation WebSockets, and many Cloudflare-specific services need replacement. | Do not select without a separate user-approved design if the spike fails. |
| Separate local API implementation | Could be purpose-built for local desktop. | Duplicates business rules, auth, chat persistence, and provider selection; likely to drift from hosted behavior. | Rejected unless a bounded set of local-only use cases proves reuse impossible. |

### 3. Local workspace and agent execution

**Decision**: Require the user-selected Docker Desktop + WSL2 Linux-container runtime. Add a local lifecycle adapter and prove the Go VM Agent integration; do not infer that existing standalone mode already provides this feature.

**Evidence and rationale**:

- Hosted workspace creation resolves cloud providers and creates remote nodes/VMs; the VM Agent runs on the VM host and manages workspace containers there.
- `apps/api/src/durable-objects/vm-agent-container.ts` and the `VM_AGENT_CONTAINER` binding in `apps/api/src/env.ts` refer to Cloudflare Container execution, not Docker Desktop.
- `packages/vm-agent/main.go` standalone mode is explicitly for one Cloudflare Container and skips host provisioning, Docker, devcontainers, TLS, DNS, and port scanning.
- `packages/vm-agent/internal/acp/process.go` defines `LocalLauncher` as a process launcher. `packages/vm-agent/internal/server/server.go` selects it for standalone mode. This is reusable process/ACP logic, not local workspace provisioning or lifecycle management.
- `packages/vm-agent/AGENTS.md` documents that nested containers are a product requirement and that the node VM, not the devcontainer, is the security boundary.
- Docker Desktop runs the Linux Engine inside its VM. It supports the special `host.docker.internal` name for reaching host services; published ports default to all interfaces unless explicitly bound. Docker's documentation says host firewall controls can filter traffic through `com.docker.backend.exe`.

**Alternatives considered**:

| Option | Benefits | Costs / risks | Assessment |
|---|---|---|---|
| Local Docker driver with a trusted Go runner/VM Agent path | Reuses ACP/session contracts and the user's approved Docker isolation model. | Requires a new local lifecycle path. Preserve supported repository devcontainer and nested-Docker behavior; treat Docker Desktop's WSL2 Linux VM as the security boundary and do not publish its Docker API beyond that boundary. | Recommended direction, gated on a real end-to-end prototype that verifies the existing trust boundary. |
| Reuse Cloudflare Container DO locally | Minimal apparent API changes. | Cloudflare Container DOs are Cloudflare runtime bindings and do not manage Docker Desktop workspaces. | Rejected. |
| Run agents directly on Windows | Avoids Docker Desktop networking. | Does not satisfy the isolation requirement and the current VM Agent process/session path is Linux-oriented. | Rejected. |

### 4. Local and cloud model routing

**Decision**: Make inference mode explicit and route local inference through a SAM-owned model gateway; preserve cloud as an independent explicit mode.

**Evidence and rationale**:

- `packages/shared/src/types/agent-settings.ts` already models OpenCode providers (`opencode-zen`, `opencode-go`, and `custom`) and stores a custom base URL.
- `apps/api/src/schemas/agent-settings.ts` currently accepts only HTTPS for `opencodeBaseUrl`. Do not weaken this rule for hosted/cloud settings. A local mode needs a separately validated local endpoint contract.
- `packages/vm-agent` passes agent configuration and environment variables to ACP processes. Agent selection must be tested through this real launch path, not only by saving a UI setting.
- Docker Desktop documents `host.docker.internal` as a route to the host. This establishes reachability, not LAN isolation or authorization. The model endpoint must not be opened on all interfaces as a workaround.
- The current agent settings also include SAM-managed proxy and user/OAuth modes. SAM-managed inference depends on the hosted SAM service and must not be assumed available to an offline local installation.

**Proposed routing invariant**:

1. Persist one explicit mode (`local` or `cloud`) for each supported agent.
2. In `local` mode, inject only a short-lived workspace-scoped token and the SAM gateway URL. The gateway resolves a saved connection reference; the agent receives no stored upstream credential and cannot choose an arbitrary upstream URL.
3. In `cloud` mode, use only the selected existing cloud provider and its configured credential. Failure is surfaced; no local fallback occurs.
4. Do not send a prompt to the SAM-hosted `sam` proxy unless the user explicitly selects that hosted dependency.

### 5. Data ownership, authentication, and upgrades

**Decision**: Treat local mode as a separate single-user installation, not a client-side variation of hosted auth.

**Evidence and rationale**:

- `apps/api/src/auth.ts` configures BetterAuth with Cloudflare D1/KV and OAuth. A local install cannot require a Cloudflare account or hosted OAuth callback.
- Existing routes perform server-side ownership checks and VM callbacks use scoped tokens/JWTs. Local mode must retain those server-side checks; a desktop renderer's request is not authorization by itself.
- Persist application metadata under the Windows user's application-data directory. Keep source repositories and Docker resources separately addressable; deletion of one category must not implicitly delete another.
- Local credentials should live in Windows user-protected storage with references in database rows. Electron safeStorage documents DPAPI protection between Windows users but not against other processes running as the same user. Do not persist secrets in logs or D1 as plaintext.
- A new local install has no hosted database to migrate. It should bootstrap a single local principal and run versioned migrations for its own data. Upgrades must back up/verify app-owned database state before schema changes.

### 6. Failure modes and release gates

- If local Worker bindings, migrations, or persistent DO state cannot run from a packaged install without remote credentials, stop before implementation and seek approval for a Node/SQLite replacement.
- If Docker Desktop/WSL2 is unavailable or the runtime driver cannot confirm cleanup, return an explicit error and do not use a remote VM.
- If the private Docker-to-model route cannot be proven isolated from LAN access, do not ask users to bind Ollama to `0.0.0.0`; redesign the broker and retest.
- If a local model connection fails, do not send prompts to cloud. If cloud selection fails, do not send prompts to local.
- If an app exits during active work, reconcile only SAM-labeled resources and report states; never use global Docker cleanup.
- If the user chooses background operation, keep the desktop supervisor visibly running; default close behavior remains graceful stop.

## Primary References

### Repository evidence

- `apps/api/src/index.ts` — API route assembly, including `/api/workspaces`, `/api/agent-settings`, and project chat routes.
- `apps/api/src/env.ts` and `apps/api/wrangler.toml` — required Worker bindings, Durable Objects, migrations, and triggers.
- `apps/api/src/auth.ts` — Cloudflare-backed BetterAuth setup.
- `apps/api/src/durable-objects/project-data/index.ts` — ProjectData Durable Object and embedded SQLite role.
- `apps/api/package.json` — `wrangler dev` startup and declared Node adapter; no Node API startup script.
- `apps/www/src/content/docs/docs/guides/local-development.md` — current developer-only local limitations (no agent sessions or VM provisioning).
- `apps/www/src/content/docs/docs/architecture/overview.md` — hosted Worker/DO/VM data path.
- `packages/vm-agent/main.go`, `packages/vm-agent/internal/acp/process.go`, and `packages/vm-agent/internal/server/server.go` — standalone mode and process launcher scope.
- `packages/shared/src/types/agent-settings.ts` and `apps/api/src/schemas/agent-settings.ts` — current OpenCode provider settings and HTTPS validation.
- `packages/vm-agent/AGENTS.md` — current node-VM trust boundary and nested-container requirements.

### Official external references

- [Cloudflare Workers local development](https://developers.cloudflare.com/workers/local-development/) — Miniflare/Workerd execution, local bindings, and the warning that local execution is not a hardened sandbox.
- [Cloudflare D1 local development](https://developers.cloudflare.com/d1/best-practices/local-development/) — local D1 environment behavior.
- [Cloudflare adding local data](https://developers.cloudflare.com/workers/local-development/local-data/) — local binding state under `.wrangler/state`, local data separation, and Durable Object population considerations.
- [Cloudflare choosing Wrangler and Vite](https://developers.cloudflare.com/workers/local-development/wrangler-vs-vite/) — both use Miniflare for local binding simulations.
- [Docker Desktop networking](https://docs.docker.com/desktop/features/networking/) and [networking how-tos](https://docs.docker.com/desktop/features/networking/networking-how-tos/) — host/container networking, `host.docker.internal`, port publishing, and firewall visibility.
- [Electron security recommendations](https://www.electronjs.org/docs/latest/tutorial/security) and [safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage) — renderer isolation and Windows DPAPI semantics.
- [Tauri security model](https://v2.tauri.app/security/) and [getting started](https://v2.tauri.app/start/) — capability-based frontend IPC and system-WebView tradeoffs.
