# Native Local SAM Desktop

## Problem

SAM currently runs as a Cloudflare-hosted control plane that provisions remote VM workspaces. Its documented local development mode is for contributors: it does not support agent sessions or workspace provisioning. Users need a native Windows desktop version that runs SAM and its agent workspaces on their own machine, and lets them choose local models or cloud models per agent.

## User-Confirmed Scope

- First release targets Windows; additional desktop platforms can follow.
- SAM itself must run locally as a native desktop application, not only as a browser app connected to the hosted control plane.
- Agent workspaces and the control plane run locally; remote VM provisioning is not the local mode.
- Users can choose a local model or a cloud model per agent.
- Local workspaces require Docker Desktop with WSL2 Linux containers.
- OpenCode is the first local-inference path; Ollama is the first verified OpenAI-compatible local model service.
- Isolated workspaces need protected host-model access without exposing the service to the LAN.
- Closing SAM during active work asks whether to stop or keep work in the background; graceful stop is the default.

## Research Findings

- `apps/www/src/content/docs/docs/guides/local-development.md` says the current Wrangler/Vite local development setup does not support agent sessions or VM provisioning.
- `specs/002-local-mock-mode/` describes an earlier, developer-only mock mode with in-memory state and a single local devcontainer. It does not describe a native, persistent, end-user desktop product.
- The API is a Cloudflare Worker and binds D1, KV, R2, and many Durable Objects in `apps/api/wrangler.toml`; a local desktop release must define how these services run and persist without a Cloudflare account.
- The existing workspace architecture places the Go VM Agent on a cloud VM and runs workspace containers under its Docker Engine. Local workspaces therefore need a supported local execution/agent path.
- `LocalLauncher` in `packages/vm-agent/internal/acp/process.go` starts an agent process; it does not provision or manage a local workspace. The current standalone mode is for Cloudflare Containers and skips Docker/devcontainer setup.
- `packages/vm-agent/AGENTS.md` treats nested containers as a product requirement and the node VM—not the devcontainer—as the security boundary. Local work must preserve supported repository devcontainer behavior within Docker Desktop's WSL2 Linux VM and must not expose its Docker API to the LAN.
- OpenCode already has per-agent provider settings and a custom OpenAI-compatible endpoint, but `apps/api/src/schemas/agent-settings.ts` currently requires HTTPS. Local inference needs an explicit local mode and separate endpoint validation; hosted validation must remain unchanged.
- Cloudflare documents Wrangler/Miniflare local bindings and persistence for development, but not a packaged desktop runtime. A local Worker-runtime feasibility test is required before selecting this as the product runtime.
- Docker Desktop documents `host.docker.internal`; this proves a container-to-host route, not LAN isolation. A gateway, scoped authorization, and real Windows firewall/network tests are required.
- The existing setup is multi-service and Cloudflare-specific. This is a new cross-component architecture, so create and review a feature spec and implementation plan before changing runtime code.

## Architecture Recommendations And Approval Gates

- Recommend a packaged desktop shell reusing `apps/web`; Electron is the effort-minimizing candidate, while Tauri offers a smaller capability-oriented shell but introduces Rust.
- First test whether a pinned Workerd/Miniflare runtime can preserve the existing Hono/D1/ProjectData contracts with durable app-owned state. Do not ship the development runtime by assumption; if it fails, stop and request approval for a Node/SQLite design.
- Add a local Docker workspace driver and reuse Go VM Agent ACP/session behavior only where a real local lifecycle path proves compatible. Preserve supported devcontainer and nested-Docker behavior within the Docker Desktop/WSL2 trust boundary; do not expose the Docker API to the LAN.
- Route local-model requests through a SAM-owned gateway with a workspace-scoped token. Do not tell users to bind Ollama to all host interfaces; failure to prove a private path blocks implementation.
- Use Windows user-protected credential storage with references, not secrets, in the local database. Confirm the selected shell/store and clarify same-user process exposure before implementation.
- Confirm whether local projects start from user-selected folders, whether GitHub remotes are needed in the first release, and whether the hosted `sam` inference proxy is in scope for local installs. Preserve the established devcontainer/nested-Docker requirement rather than treating it as an unresolved scope choice.

## Implementation Checklist

- [x] Create and clarify the feature specification with the user's confirmed Windows, Docker/WSL2, local-model access, and close behavior decisions.
- [x] Map the current API, storage, Durable Object, UI, and VM Agent paths; document the local Worker-runtime feasibility gate.
- [x] Plan local data ownership, authentication/security boundaries, service lifecycle, Docker integration, model routing, failure modes, and verification gates. The user approved the architecture plan for task generation; implementation remains gated on feasibility results, shell/credential choices, and product-scope decisions.
- [x] Generate dependency-ordered Prompt 5 implementation tasks in `specs/036-local-desktop/tasks.md`; no application code was changed.
- [x] Complete Prompt 6 consistency analysis and apply the user-approved spec, plan, task, data-model, and API-contract corrections; no application code was changed.
- [ ] Implement the Windows native desktop entry point and local service startup/shutdown with clear prerequisite/error reporting.
- [ ] Implement persistent local control-plane storage and local workspace/agent execution without requiring Cloudflare or remote VM provisioning.
- [ ] Add per-agent local-model configuration and an end-to-end model call through a local endpoint; retain and test cloud-model selection.
- [ ] Add focused behavioral and vertical-slice tests for startup, workspace creation, model switching, local inference, persistence, and cleanup.
- [ ] Update user documentation, setup guidance, security boundaries, and troubleshooting for the local desktop release.
- [ ] Validate the packaged app on Windows with Docker and a real local model endpoint; verify cloud-model selection separately.

## Prompt 4 Planning Outcome

- Added `specs/036-local-desktop/plan.md`, `research.md`, `data-model.md`, `contracts/local-desktop.openapi.yaml`, and `quickstart.md`.
- No application code or hosted behavior was changed.
- The user approved the architecture plan for task generation. Implementation remains blocked on proving the packaged local Worker/storage and private model-network paths and recording the shell/credential and product-scope choices.

## Prompt 5 Task Generation Outcome

- Added `specs/036-local-desktop/tasks.md` with 40 executable tasks grouped by setup, foundation, the four user stories, and cross-cutting verification.
- The first full Windows -> local workspace -> OpenCode -> Ollama streamed-response vertical slice is scheduled in User Story 2, after the three feasibility gates.
- Prompt 5 is complete; no product implementation has started.

## Prompt 6 Analysis Outcome

- The initial mapping linked all 23 functional requirements and 9 success criteria to at least one of the 40 tasks, but identified missing constitution release gates, incomplete offline/five-minute verification, underspecified cloud credential setup and local-model configuration contract, and gaps around per-agent isolation, disk-full handling, and packaged uninstall behavior.
- Applied the user's approval to correct all findings: added explicit product-scope approval gates, strengthened requirements/tests and coverage checks, clarified task boundaries, and specified local model connection configuration/verification operations.
- Prompt 6 is complete. The technical feasibility gates and product-scope owner decisions still block implementation; no product implementation has started.

## Acceptance Criteria

- A Windows user can install and launch SAM as a native desktop application without configuring a Cloudflare account.
- SAM's local application data and agent workspaces persist across app restarts and can be removed through documented cleanup.
- A user can create a local workspace, start an agent, and observe its response in SAM without provisioning a remote VM.
- Per-agent settings allow selecting a reachable local model endpoint or an existing cloud model provider; changing the selection changes the model endpoint actually used by the agent.
- Local-model requests remain on the configured local endpoint; no local prompt or credential is silently sent to a different provider.
- Missing Docker, an unreachable local model endpoint, and invalid settings produce actionable errors without leaving orphaned workspaces or success-shaped failures.
- Local app services are not exposed to the network by default, and locally stored credentials are protected.
- Supported repository devcontainer and nested-Docker workflows remain usable without exposing the Docker API beyond Docker Desktop's WSL2 Linux VM.
- A tested Windows install-to-first-agent workflow demonstrates the complete local path, including a real local-model response.

## References

- `apps/www/src/content/docs/docs/guides/local-development.md`
- `apps/www/src/content/docs/docs/architecture/overview.md`
- `specs/002-local-mock-mode/`
- `apps/api/wrangler.toml`
- `packages/vm-agent/`
- `packages/shared/src/types/agent-settings.ts`
- `.specify/memory/constitution.md`
