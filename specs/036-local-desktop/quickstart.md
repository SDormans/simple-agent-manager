# Local Desktop Architecture Validation Quickstart

This is a planning-stage verification checklist, not a claim that the packaged local desktop exists. Complete and record these checks before implementation is approved.

## Required Environment

- Supported Windows version with WSL2 and Docker Desktop using Linux containers.
- A clean SAM checkout with the repository's existing Node/pnpm and Go toolchains.
- Ollama installed on the Windows host with a small test model available.
- A second device on the local network for negative network-exposure checks.
- No Cloudflare account credentials in the test environment.

## Feasibility Sequence

1. **Worker runtime**: Start the pinned local Worker runtime with all bindings local. Confirm it does not attempt remote bindings or require Cloudflare secrets.
2. **Durability**: Create a project and chat record, stop and restart the runtime, and confirm D1 and ProjectData Durable Object state remains intact in an app-owned data path.
3. **Migrations**: Run the full migration chain against an empty local store, upgrade a populated store, verify a pre-upgrade backup, and prove a failed migration preserves the prior copy.
4. **Local mode isolation**: Exercise project/chat paths with network egress instrumentation. Confirm no Cloudflare API, Workers AI, DNS, provider VM, or cron path is called.
5. **Docker lifecycle**: Through the proposed desktop control path, create, start, stop, restart, and delete one SAM-labeled workspace. Verify supported repository devcontainer and nested-Docker behavior. Restart the app mid-run and reconcile only that workspace; verify no unrelated Docker resource changes.
6. **Model bridge**: From the workspace container, reach the configured Ollama model through the SAM gateway. Verify the gateway cannot be pointed at an arbitrary host/URL and that a second LAN device cannot reach the UI, API, gateway, or model service.
7. **Routing controls**: Instrument both local and cloud endpoints. A local-mode prompt must hit only the local endpoint; a cloud-mode prompt must hit only the configured cloud endpoint; unavailable endpoints must fail without reaching the other endpoint.
8. **Desktop lifecycle**: Package the shell, launch it on a clean Windows profile, test the default graceful-stop prompt, explicitly choose background mode, and verify visible running status and subsequent shutdown behavior.
9. **Data cleanup**: Remove a model credential and delete a workspace while verifying project/conversation data and unrelated Docker resources remain. Exercise uninstall retain/delete choices separately.

## Stop Conditions

- Stop the local-runtime path if it cannot support persistent D1/DO state and migrations without Cloudflare credentials or remote bindings.
- Stop if local control or model ports can be reached from a LAN device by default.
- Stop if the local path cannot preserve supported repository devcontainer and nested-Docker behavior within Docker Desktop's WSL2 Linux VM security boundary, or requires publishing the Docker API beyond that boundary. Seek owner approval before changing the established trust model.
- Stop if a local-mode failure causes a cloud/remote-VM fallback or a cloud-mode failure causes local routing.
- Do not substitute Miniflare unit tests or a page-load check for the packaged Windows workspace-to-agent-to-model path.

Record Windows/build/runtime versions, exact commands, endpoint-routing evidence, Docker resource IDs/labels, persistence results, and cleanup outcomes. Any failed stop condition requires an architecture decision before implementation continues.
