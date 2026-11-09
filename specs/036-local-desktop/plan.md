# Implementation Plan: Native Local SAM Desktop

**Branch**: `036-local-desktop` | **Date**: 2026-11-02 | **Spec**: [spec.md](./spec.md)
**Input**: Clarified feature specification in `specs/036-local-desktop/spec.md`

## Summary

Deliver a Windows single-user SAM desktop that packages the existing React control plane, runs a local API and durable local state, manages isolated Docker Desktop/WSL2 workspaces, and routes each supported agent explicitly to either a local model gateway or its configured cloud provider.

**Architecture recommendation approved for task generation, not implementation approval:** evaluate a narrowly privileged Electron shell to reuse the existing React/Vite UI and supervise local services; first prove that a pinned Miniflare/Workerd runtime can run the trusted API with persistent local bindings and no Cloudflare account; add a local workspace driver around Docker Desktop and reuse the Go VM Agent's ACP/session behavior where its runtime contracts fit. A SAM-owned model gateway should broker local inference and keep model-service credentials out of agent containers. Hosted deployment behavior remains on its current Cloudflare/VM path.

This is deliberately conditional: Cloudflare documents Miniflare/Wrangler local execution as a development environment, not as a packaged desktop product. Do not ship it as the product runtime until the feasibility tests in [quickstart.md](./quickstart.md) pass. If they fail, stop and return a Node/SQLite adapter design for owner approval rather than silently porting or duplicating the API.

## Technical Context

**Language/Version**: Existing TypeScript 5.x and Go 1.26.6; desktop shell/runtime language is gated on T001-T003 and owner approval.
**Primary Dependencies**: Existing React/Vite, Hono, Wrangler/Miniflare, and Go VM Agent; desktop shell package is gated on T001-T003 and owner approval.
**Storage**: Existing D1/KV/R2/ProjectData Durable Object data contracts; packaged runtime and persistent local backend are gated on T001-T003 and owner approval.
**Testing**: Vitest, Go tests, Worker/Miniflare tests, and a packaged Windows end-to-end suite with Docker Desktop and Ollama
**Target Platform**: Windows first; one local OS user; Docker Desktop with WSL2 Linux containers
**Project Type**: Existing monorepo web/API applications plus a proposed desktop shell and local runtime adapter
**Performance Goals**: Startup and first local project creation within the spec's five-minute target; streaming remains responsive; no measurable local-model request is redirected to a cloud provider
**Constraints**: No Cloudflare account or remote VM required for local mode; local state survives restarts/upgrades; UI listener is loopback-only; workspace access uses an authenticated private path; no implicit inference fallback; no changes to hosted behavior
**Scale/Scope**: Single Windows user, local projects/workspaces, OpenCode as the first local-inference agent, Ollama as the first verified local service, and existing explicitly selected cloud-provider paths

### Planning Assumptions Requiring User Review

- Local folders are the primary project source; Git hosting integrations are optional and require explicit credentials/network access.
- The local desktop does not use the platform-managed `sam` inference proxy unless the user explicitly approves an opt-in hosted dependency. Direct configured cloud providers remain a separate explicit choice.
- Existing project data is not imported from a hosted SAM account. A fresh local install bootstraps its own single local principal and replays its own migrations.
- Workspaces and repository files are never removed by application-data cleanup unless the user separately confirms that specific deletion.
- A selected local model endpoint is reached through a SAM-owned gateway; agent containers receive a short-lived, workspace-scoped gateway credential, not the model-service credential or arbitrary upstream URL.

### Product Scope Approval Gate

Before T004, record the owner's decision on both unresolved product choices:

1. Whether first-run project creation is local-folder-first, with Git remote cloning remaining a separately explicit action.
2. Whether the platform-managed `sam` inference proxy is excluded from local installations or is an explicitly supported cloud destination.

Do not treat the second choice as settled merely because direct cloud-provider credentials remain supported. If no decision is available, keep implementation blocked rather than silently narrowing the cloud-model options.

## Constitution Check

| Principle | Result | Plan evidence / gate |
|---|---|---|
| I. Open Source Sustainability | Pass | Desktop and local-runtime components remain in the open-source monorepo. |
| II. Infrastructure Stability | Gate | T040 must report >90% coverage for local authentication/ownership, workspace lifecycle/cleanup, persistence/migration/recovery, credential storage, and inference routing/gateway, plus >80% overall; critical-path work uses tests-first sequencing and behavioral/vertical-slice coverage before release. |
| III. Documentation Excellence | Pass for plan; implementation required | Add a public Windows local-desktop guide and update architecture/security/local-development docs with verified behavior and code paths. |
| IV. Approachable Code & UX | Pass | Missing prerequisites, provider failures, and cleanup outcomes must be explicit and actionable. |
| V. Transparent Roadmap | Pass | Keep the feature spec and dependency-ordered implementation tasks as the durable roadmap for this work. |
| VI. Automated Quality Gates | Pass | Reuse existing TypeScript/Go checks; add Windows package and Docker-backed integration coverage. |
| VII. Inclusive Contribution | Pass | Document Windows prerequisites and local setup so users and contributors can follow the supported path. |
| VIII. AI-Friendly Repository | Pass | Keep the architecture and task boundaries documented in the feature spec for future contributors and agents. |
| IX. Clean Code Architecture | Gate | Keep local orchestration behind a runtime boundary; do not fork hosted routes or add a second implementation without a concrete compatibility reason. |
| X. Simplicity & Clarity | Gate | Electron/Miniflare are recommendations only; prove they reduce total complexity before adding dependencies or treating a development runtime as a product runtime. |
| XI. No Hardcoded Values | Pass with implementation check | Ports, timeouts, path locations, resource limits, and service addresses need typed defaults/configuration; T040 audits new settings and defaults against this requirement. |
| XII. Zero-to-Production Deployability | Pass | No hosted infrastructure is required for local mode; hosted deployment and migration behavior remain unchanged. |
| XIII. Fail-Fast Error Detection | Pass | Local runtime, Docker, model, and cleanup errors fail visibly; no cloud/local fallback or success-shaped cleanup response. |

**Pre-research gate**: No constitutional violation is required. The major technology and network-boundary decisions remain user approval gates.
**Post-design gate**: Conditional until the local-runtime feasibility spike, credential-storage decision, product scope choices above, and private Docker-to-model path are approved and demonstrated. No implementation may weaken hosted identity/ownership guards or treat the desktop client as trusted authorization.

## Target Data Paths

### Local inference

```text
Packaged React UI
  -> authenticated local control API
  -> local project/agent settings and durable ProjectData storage
  -> local workspace driver
  -> Docker Desktop / WSL2 isolated workspace
  -> Go VM Agent ACP/session path
  -> SAM-owned local model gateway on the private workspace network
  -> configured Ollama/OpenAI-compatible host service
  -> streamed response persisted to the matching project conversation
```

The gateway must be reachable by the workspace over a private Docker path, not by arbitrary LAN clients. The exact Windows/Docker interface and firewall behavior is a release-blocking feasibility test; do not configure Ollama to listen on all host interfaces as a shortcut.

### Cloud inference

```text
Packaged React UI
  -> local control API and per-agent settings
  -> local workspace / Go VM Agent
  -> explicitly selected cloud provider endpoint
  -> streamed response persisted to the matching project conversation
```

The cloud path is independent of local-mode selection. An unavailable endpoint is an error; no request may switch destinations. The SAM-hosted `sam` proxy is not assumed to be available from a local installation.

## Project Structure

### Documentation (this feature)

```text
specs/036-local-desktop/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── local-desktop.openapi.yaml
└── tasks.md              # Prompt 5 output
```

### Proposed Source Structure (subject to approval)

```text
apps/
├── desktop/              # Windows shell, lifecycle supervisor, packaged UI
├── api/                  # Existing Hono Worker; local-mode boundary/adapters
└── web/                  # Existing React/Vite control plane, packaged unchanged where possible
packages/
├── shared/               # Explicit per-agent inference-mode contract
└── vm-agent/             # Existing ACP/session code plus a separately gated local runtime path
```

**Structure Decision**: Add only a desktop package and narrow local runtime adapters if the feasibility spike validates the Worker/local-binding approach. Keep the existing hosted API, provider selection, and workspace lifecycle as the hosted implementation. Do not create a second local API business-logic tree.

## Design and Compatibility

1. **Desktop shell**: Electron is the implementation-effort recommendation because the UI is already React/Vite and a desktop main process can supervise child services. Keep Node disabled in the renderer, enable context isolation and sandboxing, load only packaged local assets, use a restrictive CSP, and expose only narrow typed IPC. Tauri is the alternative if the owner prioritizes a smaller native shell and capability-oriented IPC over reusing the TypeScript process layer.
2. **Control plane and storage**: Time-box a proof that the existing Hono Worker and ProjectData Durable Objects run on the pinned local Workerd/Miniflare path with D1/KV/R2/DO state persisted in the app-owned data directory. Disable remote bindings, Workers AI, cron/scheduled work, and Cloudflare-only operations in local mode. The local runtime is trusted SAM code, never a sandbox for project or agent code.
3. **Workspace runtime**: Add a local Docker Desktop driver; do not route local mode through cloud-provider resolution, cloud-init, DNS, VM provisioning, or Cloudflare Container Durable Objects. Preserve supported repository devcontainer settings, including nested-Docker workflows. Treat Docker Desktop's WSL2 Linux VM as the security boundary, not the devcontainer; do not expand Docker access or publish the Docker API beyond that boundary. Reuse VM Agent ACP/session behavior only after a concrete local launch path is tested. `LocalLauncher` is a child-process launcher used by standalone mode; it is not a local workspace manager.
4. **Authentication and credentials**: Create a single local principal and server-validated local session. Preserve project/workspace ownership checks and callback-token scoping. Keep provider secrets outside D1 in a Windows user-protected store; D1 stores references and non-secret settings only. Credential create/update/remove must use the protected-store boundary; secrets must not be persisted in D1, renderer storage, workspaces, or logs.
5. **Inference**: Persist an explicit `local` or `cloud` choice per supported agent. Use the existing OpenCode custom OpenAI-compatible provider shape as a starting point, but do not weaken the hosted HTTPS validator globally. Add a local-only, bounded endpoint contract and gateway; reject unsupported schemes/hosts and arbitrary proxy targets. No automatic fallback.
6. **Networking**: Serve the desktop UI/control interface on loopback. Keep workspace callbacks and the model gateway on a private Docker network with scoped tokens. Verify a second LAN device cannot reach either listener or Ollama. Docker Desktop's `host.docker.internal` provides a host route, but by itself does not prove a service is LAN-isolated.
7. **Data and upgrades**: Persist the local database under the per-user application data directory; keep repository paths and Docker workspace resources separately identified. Back up/version local database state before migrations. Integrate the retain/delete choice with the packaged Windows uninstall flow before data deletion; do not rely on the app main process running after uninstall. Uninstall retains data by default and offers explicit, scoped deletion; never run global Docker prune.
8. **Hosted compatibility**: Runtime mode is decided by the desktop installation, not a client request. New local behavior is gated on local-mode server configuration. Hosted routes, auth, secrets, and provider resolution retain their current defaults and receive regression tests.

## Required Verification Before Implementation Approval

- Prove packaged startup, persistent local D1/KV/R2/DO state, and migration/restart behavior with no Cloudflare account or remote binding.
- Prove the API does not invoke Cloudflare APIs, Workers AI, remote VM providers, or scheduled cleanup during local-only workflows.
- Prove workspace create/start/stop/restart/delete through the actual desktop control path, preserve supported repository devcontainer and nested-Docker behavior, and confirm cleanup by inspecting Docker resources.
- Prove workspace-to-model gateway access on Windows/WSL2 while an unrelated LAN device cannot reach the local UI, API, gateway, or model service.
- After installation and downloading required local models/images, disable outbound internet and prove a complete local project-to-agent-response flow still works.
- Prove local inference reaches only the configured local service and cloud inference reaches only the selected cloud provider, with the opposite endpoint instrumented as a negative control.
- Configure two supported agents with different inference destinations and prove each uses only its own saved selection.
- Run a real local Ollama request and a real supported cloud-provider request in the packaged app; mocks are not sufficient for release readiness.
- Measure first-run launch-to-project creation against the five-minute success criterion on a documented supported Windows baseline.
- Run hosted control-plane/provider/workspace regression tests to prove no hosted behavior changed.

## Complexity Tracking

| Proposed addition | Why it may be needed | Simpler alternative rejected because |
|---|---|---|
| Desktop shell | Required for a native Windows install, service supervision, and close/background behavior. | Browser-only Vite mode does not meet the user requirement or provide reliable process lifecycle control. |
| Local runtime adapter | Required to keep local workspace execution separate from cloud VM provisioning. | Reusing cloud providers would silently provision remote resources; copying all business logic would fork behavior. |
| Model gateway | Required to protect host-local model access and prevent agent containers from receiving provider credentials or arbitrary upstream access. | Direct `host.docker.internal` access does not prove LAN isolation or constrain the target. |
| Worker local-runtime proof | May preserve API/ProjectData behavior without a broad storage rewrite. | A Node/SQLite rewrite must replace many Cloudflare bindings and Durable Object contracts; only consider it if the local Worker path fails verified requirements. |
