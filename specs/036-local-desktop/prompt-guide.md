# Prompt Guide: Native Local SAM Desktop

Use these prompts in order to take the local-desktop idea from discovery to a
tested implementation. Run one prompt at a time and review its result before
continuing. The prompts define the work; they do not authorize skipping the
repository's normal quality, security, or review gates.

## Confirmed product direction

- SAM itself runs as a native desktop application on the user's computer.
- The first desktop release targets Windows.
- The control plane, saved application data, agent execution, and workspaces run
  locally; a Cloudflare account and remote VM are not prerequisites.
- Local workspaces require Docker Desktop with WSL2 Linux containers.
- Each supported agent can be configured to use local inference or cloud
  inference.
- OpenCode is the first local-inference path. The first verified local service
  is Ollama through an OpenAI-compatible API.
- An isolated workspace must reach the host-local model service without
  exposing that service to the local network.
- Closing SAM during active work asks whether to stop or keep running in the
  background; graceful stop is the default.

## How to use

1. Use one branch per prompt step. Name implementation branches
   `036-local-desktop-prompt-<number>-<short-description>` and base each new
   branch on the latest `main` after the previous prompt's PR has merged. Do
   not continue work on a branch used for an earlier prompt.
2. Paste the prompts below one at a time, replacing bracketed values when
   needed.
3. For each prompt, create its branch before starting. After a step that
   changes files, commit and push that branch, open a PR to `main`, and stop
   for the user's review and approval. Merge the approved PR before creating
   the next prompt's branch. A read-only prompt may use its own branch but
   needs no commit or PR if it changes no files.
4. Do not start implementation until the user approves the spec, architecture,
   security boundaries, and local runtime feasibility gates.
5. If a tool rejects a file edit, stop and ask before retrying the same edit.
6. Keep the work single-user and local-first unless the product owner expands
   the scope.

## Prompt 1 — Research the existing system (read-only)

```text
Research how to deliver the confirmed Native Local SAM Desktop direction
without changing files.

Trace the real code paths and report evidence for:
- desktop/web entry points and current app packaging;
- API startup and every Cloudflare-specific binding or service used by the
  local agent/workspace/chat path (D1, KV, R2, Durable Objects, Workers AI,
  queues, cron, auth, and WebSockets as applicable);
- local development and mock-mode behavior, including what demonstrably does
  not work;
- workspace creation, agent process startup, ACP/chat streaming, and cleanup;
- model/provider selection and where OpenCode obtains and calls its inference
  endpoint;
- secrets, local data persistence, and network exposure boundaries.

Inspect the current implementation and its tests; do not infer working
behavior from old specs or documentation alone. Identify which existing pieces
can be reused, which depend on Cloudflare or remote VMs, and what must change.
For each claim, cite file paths and relevant symbols. End with the unresolved
product/architecture decisions that must be answered before planning. Do not
implement, scaffold a desktop shell, or rewrite existing files.
```

## Prompt 2 — Write the feature specification

```text
Use /speckit.specify for a feature named Native Local SAM Desktop.

Use the confirmed product direction in specs/036-local-desktop/prompt-guide.md
and the verified research findings from the previous step. Specify user-visible
outcomes only; do not choose frameworks, databases, IPC protocols, or a desktop
packager in the spec.

The spec must cover:
- Windows native installation, first run, start/stop, upgrade, and uninstall;
- local control-plane operation and durable local projects, settings, and chat;
- running and cleaning up an agent workspace on the same machine without a
  remote VM;
- per-agent local-vs-cloud inference choice, initially OpenCode for local
  inference and existing cloud paths where compatible;
- visible provider selection, explicit errors, and no silent local-to-cloud
  fallback;
- offline/local-only use, crash recovery, data cleanup, credentials, and
  loopback-only service exposure by default.

Bound the feature to a single local user and Windows first. Exclude hosted
sync, multi-user access, remote VM provisioning in local mode, and additional
desktop OSes unless research proves one is necessary. Turn unknown user-facing
decisions into a short, prioritized clarification list rather than silently
assuming them. Write the spec and its quality checklist only; do not edit
application code or other feature specs.
```

## Prompt 3 — Clarify product decisions (completed)

```text
The user has already confirmed the following decisions in the spec:
- Require Docker Desktop with WSL2 Linux containers.
- Support OpenAI-compatible local model services; verify first with Ollama.
- Allow protected access from isolated workspaces to the host-local model
  service without LAN exposure.
- When closing SAM during active work, ask whether to stop or continue in the
  background; default to graceful stop.

Do not ask these questions again. The clarification step is complete. Any new
product decisions discovered during Prompt 4 must be listed as owner approval
gates and reviewed before implementation; do not silently add them to the spec.
```

## Prompt 4 — Design the architecture (completed; pushed as `7bcad4632`)

```text
Use /speckit.plan for the clarified Native Local SAM Desktop spec.

First map the end-to-end target data paths: desktop launch -> local API ->
durable local storage -> local workspace runtime -> agent process -> model
endpoint -> streamed chat response. Separately map the cloud-model path.

Compare the smallest viable architecture options for:
- a native Windows shell and supervised local services;
- replacing or adapting Cloudflare-only API/storage/DO dependencies for
  persistent single-user local operation;
- starting, monitoring, recovering, and stopping local isolated workspaces;
- preserving supported repository devcontainer behavior, including
  nested-Docker workflows, within Docker Desktop's WSL2 Linux VM security
  boundary;
- reaching a host-local model server from an agent inside an isolated workspace;
- protecting local credentials and preventing LAN exposure;
- packaging, upgrades, logs, backups, and uninstall cleanup.

Recommend one design and explain what existing code can be reused versus
isolated behind a local adapter. Do not attempt to run Cloudflare production
services locally by assumption, add speculative cross-platform infrastructure,
or create a second implementation of existing behavior without justification.
Do not make the entire API trust a desktop client. Include threat boundaries,
data ownership, failure modes, tests, migration/compatibility impact, and
documentation updates. Stop and ask me before implementation if a choice could
delete data, expose a local service, materially change the existing hosted
product, or force a major technology commitment.
```

## Prompt 5 — Generate implementation tasks (completed in this step)

```text
Use /speckit.tasks for the approved Native Local SAM Desktop spec and plan.

Create small, dependency-ordered, independently verifiable tasks. Put the
first real vertical slice early: launch the Windows app, start the local
control plane, create a local workspace, run one agent, and observe a streamed
response. Do not postpone the integration path until the end.

Include explicit tasks and tests for:
- Windows packaging and supervised service lifecycle;
- persistent local data and safe first-run initialization;
- local workspace creation, isolation, restart recovery, stop, and deletion,
  including supported devcontainer and nested-Docker behavior;
- OpenCode local endpoint configuration and cloud provider selection;
- proving each model choice reaches only its intended destination;
- credentials, loopback/network access restrictions, failure reporting, and
  cleanup;
- user documentation and a real packaged-Windows end-to-end test.

Every research finding that requires a change must become a task. Acceptance
criteria must be observable. Mark external prerequisites (Windows runner,
Docker runtime, local model server, provider credentials) explicitly. Do not
create implementation code.
```

## Prompt 6 — Check spec/plan/task consistency (completed; corrections applied)

```text
Use /speckit.analyze on the Native Local SAM Desktop spec, plan, and tasks.
Check for contradictions, missing Cloudflare-to-local replacements, unstated
security assumptions, tasks without acceptance criteria, and any user-facing
requirement that is not covered by both an implementation task and a test.

Verify that the local-model path accounts for host-versus-container
networking, that cloud inference remains an explicit separate choice, and that
no error path silently falls back to cloud. Verify first-release scope is
Windows-only and local single-user operation. Report gaps with artifact and
section references. Do not edit code; propose the smallest spec/plan/task
corrections and apply only after I approve them.
```

Prompt 6 completed read-only analysis and user-approved corrections. The
remaining implementation-blocking product decisions are explicit gates in
`tasks.md`; the technical runtime, shell/credential-store, and network gates
must still pass before product implementation.

## Prompt 7 — Implement the approved plan

```text
Use /speckit.implement to execute only the approved Native Local SAM Desktop
tasks. Follow the repository's /do workflow, create the required workflow
state, use a feature branch/worktree, and preserve unrelated user changes.

Before each code area, load its nearest AGENTS.md and routed scoped rules.
Trace existing call paths before adding helpers or abstractions. Implement in
small vertical slices; after each slice run its focused tests and update task
and workflow state. Do not replace hosted behavior for existing users: keep
the hosted and local runtime paths explicitly separated.

Never claim a local model is being used based only on a saved setting. Prove
the real agent request reaches the selected local endpoint and that the cloud
endpoint receives no request. Prove cloud selection still reaches the selected
cloud provider. Surface missing prerequisites and cleanup failures explicitly.
Do not weaken security to make loopback/container networking work. Stop and
ask me if implementation requires a scope or architecture change not approved
by the plan.
```

## Prompt 8 — Verify the actual Windows product

```text
Validate the packaged Native Local SAM Desktop as a Windows user would use it;
do not substitute page-load checks or component tests for the real workflow.

On a clean supported Windows environment:
1. Install and launch the packaged app without a Cloudflare account.
2. Complete first-run setup and create/open a local project.
3. Start an isolated workspace and run an agent through the desktop UI.
4. Configure OpenCode to use a real local model server and verify a real
   response. Capture evidence showing the local endpoint was called and the
   cloud provider was not.
5. Switch to a configured cloud model and verify its selected provider is
   called instead; verify the local endpoint is not used for that task.
6. Configure two supported agents with different destinations and verify their
   requests remain isolated to their individual selections.
7. Exercise unreachable local-model, missing workspace runtime, app restart
   during work, and workspace cleanup cases. Confirm truthful errors and no
   orphaned resources.
8. After installing prerequisites and downloading local model/workspace
   assets, disable outbound internet and verify a complete local project-to-
   agent-response flow still succeeds.
9. Close/reopen the app and verify required local state persists.
10. Confirm the local control interface is not reachable from another LAN
   device by default and confirm credentials are not exposed in logs.

Record the app version, Windows version, workspace-runtime version, local model
server/model, exact test steps, endpoint-routing evidence, and cleanup result.
Run all repository-required local checks and review gates separately. If a
real Windows package or local model endpoint is unavailable, state that
verification as incomplete; do not mark the feature ready or claim staging
proved the desktop workflow.
```

## Completion prompt — Report readiness, do not overclaim

```text
Compare the approved acceptance criteria with the actual diff, tests, Windows
package evidence, and local/cloud endpoint-routing evidence. List each
criterion as verified, failed, or not verified, with a specific test or
evidence reference. Identify unresolved blockers and reviewers. Do not archive
the task, merge, or say the feature is complete while any required criterion or
review gate is unverified.
```
