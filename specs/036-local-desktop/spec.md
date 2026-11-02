# Feature Specification: Native Local SAM Desktop

**Feature Branch**: `036-local-desktop`
**Created**: 2026-11-02
**Status**: Draft — architecture approved for task generation; implementation feasibility gates remain
**Input**: Run SAM itself as a native Windows desktop app, with the control plane, saved data, agent execution, and workspaces running locally. Let each agent use local or cloud inference; OpenCode is the first local-inference path.

## Clarifications

### Session 2026-11-02

- Q: Which Windows workspace isolation model should the first release require? → A: Require Docker Desktop with WSL2 Linux containers.
- Q: Which local model services and host/workspace access should the first release support? → A: Support OpenAI-compatible local services, verify first with Ollama, and provide protected access from the isolated workspace without LAN exposure.
- Q: When the user closes SAM while an agent is running, what should happen? → A: Ask whether to stop work or keep it running in the background, defaulting to a graceful stop.

## User Scenarios & Testing

### User Story 1 - Install and use SAM locally (Priority: P1)

As a Windows user, I want to install and launch SAM as a desktop app so I can use its control plane and create local projects without setting up a Cloudflare account or remote VM.

**Why this priority**: Running SAM itself locally is the foundation of the requested product; a model selector alone would not meet the need.

**Independent Test**: On a clean supported Windows machine, install and launch the app, complete first-run setup, create a project, close and reopen the app, and verify the project remains available without signing in to a hosted SAM service.

**Acceptance Scenarios**:

1. **Given** a supported Windows machine with Docker Desktop and WSL2 available, **When** the user installs and launches SAM for the first time, **Then** the app explains any missing prerequisite and guides the user through local setup without requiring a Cloudflare account.
2. **Given** a completed local setup, **When** the user creates a project and restarts the app, **Then** the project is still available.
3. **Given** an agent is running, **When** the user closes SAM, **Then** SAM asks whether to stop the work or keep it running in the background, defaults to a graceful stop, and clearly reports whether work remains active.
4. **Given** local setup and its prerequisites are complete, **When** the user's computer has no internet connection, **Then** the user can still open a local project and use local inference.
5. **Given** an existing hosted SAM user, **When** the local desktop capability is released, **Then** the user's existing hosted workspace and inference behavior remains unchanged.

### User Story 2 - Run an agent in a local workspace (Priority: P1)

As a user, I want SAM to create and manage an isolated workspace on my computer so I can run an agent without provisioning a remote VM.

**Why this priority**: Local agent workspaces are necessary for SAM to operate as a local program rather than only as a local dashboard.

**Independent Test**: From the desktop app, create a local workspace for a project, start an agent, observe a response in chat, stop the workspace, and verify its cleanup status.

**Acceptance Scenarios**:

1. **Given** Docker Desktop with WSL2 Linux containers is available, **When** the user starts a workspace, **Then** SAM creates an isolated workspace on the same computer and does not provision a remote VM.
2. **Given** Docker Desktop or WSL2 is unavailable, **When** the user tries to start a workspace, **Then** SAM explains what is missing and does not silently substitute a remote workspace.
3. **Given** an agent is running in a workspace, **When** it returns output, **Then** the user sees the response in the corresponding project conversation.
4. **Given** a stopped workspace, **When** the user restarts it, **Then** SAM reports whether it is ready before allowing the user to start an agent.
5. **Given** the user stops or deletes a workspace, **When** cleanup succeeds or fails, **Then** SAM reports the actual outcome and does not present an unconfirmed cleanup as successful.
6. **Given** a repository uses supported devcontainer settings for nested Docker, **When** SAM starts its local workspace, **Then** SAM preserves those settings within Docker Desktop's WSL2 Linux VM security boundary and does not expose the Docker API to the local network.

### User Story 3 - Choose local or cloud inference per agent (Priority: P1)

As a user, I want to choose local or cloud inference for each supported agent so I can keep model use on my computer when desired and retain cloud-model options.

**Why this priority**: The requested choice is per agent and must work in the agent's real request path, not only appear as a saved setting.

**Independent Test**: Configure two supported agents with different destinations, such as OpenCode using local inference and another supported agent using a cloud provider. Complete a prompt with each and verify each request reaches only that agent's selected destination.

**Acceptance Scenarios**:

1. **Given** an OpenCode agent and an OpenAI-compatible local model service, **When** the user selects local inference and configures that service, **Then** SAM saves the choice and the agent uses that local model for its requests; Ollama is the first verified service.
2. **Given** a local model connection is configured, **When** the user verifies it, **Then** SAM reports whether the connection is usable before the user starts an agent request.
3. **Given** an agent in an isolated workspace and a local model service on the Windows host, **When** the agent sends a local inference request, **Then** SAM provides protected access to the service without exposing it to the local network.
4. **Given** a supported agent with cloud inference configured, **When** the user selects cloud inference, **Then** SAM shows the selected provider and the agent uses that provider.
5. **Given** an agent with either inference mode selected, **When** the user is about to send a request, **Then** SAM makes the selected mode and destination clear.
6. **Given** an agent has a saved inference choice, **When** the user starts a later session, **Then** that choice remains associated with that agent.
7. **Given** the selected local model is unavailable, **When** the agent tries to use it, **Then** SAM presents an actionable error and does not silently send the prompt to a cloud provider.
8. **Given** the selected cloud provider is unavailable or unconfigured, **When** the agent tries to use it, **Then** SAM presents an actionable error and does not silently send the prompt to a local model.

### User Story 4 - Recover and manage local data (Priority: P2)

As a user, I want my local projects and conversations to survive ordinary restarts and to manage my workspaces, credentials, and saved data explicitly.

**Why this priority**: Local operation is only useful if the user's work is durable and cleanup is understandable and safe.

**Independent Test**: Create project and conversation data, restart the app, confirm data is restored, then remove a credential and delete a workspace while verifying project data remains unless the user explicitly deletes it.

**Acceptance Scenarios**:

1. **Given** the app exits unexpectedly, **When** the user starts it again, **Then** SAM restores saved project and conversation data and reports the state of any workspace that needs recovery.
2. **Given** the user removes a provider credential, **When** the removal completes, **Then** future use of that provider requires new credentials, unrelated project data remains, and the credential is not exposed in logs.
3. **Given** a new SAM release is available, **When** the user upgrades, **Then** saved projects, agent choices, and conversations remain available.
4. **Given** the user uninstalls SAM, **When** local application data exists, **Then** SAM clearly offers whether to retain or delete that data and confirms before irreversible deletion.

### Edge Cases

- The local workspace prerequisite is missing, stops unexpectedly, or cannot clean up a workspace.
- Docker Desktop or WSL2 is unavailable, stopped, or unable to start Linux containers.
- The configured local model server is stopped, unreachable, or reports an unsupported model.
- The protected connection from a local workspace to the host model service is unavailable.
- The user selects cloud inference without configuring its required credential or network access.
- The app loses internet access while the user is in local-only mode or while a cloud model is selected.
- The app or computer closes while an agent is running; on restart, saved data and workspace state may differ.
- The app closes unexpectedly before the user can choose whether active work should continue.
- Available disk space is insufficient to create or persist a project, conversation, or workspace.
- A request from another device on the local network attempts to reach SAM's local control interface.
- Uninstall or data deletion is interrupted; SAM must not claim data was deleted unless that result is confirmed.

## Requirements

### Functional Requirements

- **FR-001**: SAM MUST be installable and launchable as a native desktop application on Windows for one local user.
- **FR-002**: First-run setup MUST let the user start local SAM without a Cloudflare account or remote VM.
- **FR-003**: SAM MUST create, open, and retain local projects, agent settings, and conversations across normal application restarts.
- **FR-004**: SAM MUST create and manage an isolated workspace on the user's computer using Docker Desktop with WSL2 Linux containers, without provisioning a remote VM in local mode.
- **FR-005**: SAM MUST identify and explain Docker Desktop and WSL2 prerequisites before a user starts a workspace; it MUST fail visibly rather than silently changing to remote execution.
- **FR-006**: SAM MUST provide a usable chat path from the desktop app to an agent running in a local workspace and display the agent's response in the correct conversation.
- **FR-007**: For each supported agent, SAM MUST let the user select and save an inference mode independently. OpenCode MUST be the first agent with local inference support.
- **FR-008**: In local inference mode, SAM MUST let the user configure and verify an OpenAI-compatible local model service for OpenCode, with Ollama as the first verified service.
- **FR-009**: SAM MUST provide protected access from an isolated local workspace to a model service on the Windows host without exposing that service to the local network.
- **FR-010**: In cloud inference mode, SAM MUST let the user select from the supported cloud-model options and configure any required credentials.
- **FR-011**: Before a request is sent, SAM MUST make the selected inference mode and provider understandable to the user.
- **FR-012**: SAM MUST send inference requests only to the destination selected for that agent. An unavailable or invalid destination MUST produce an actionable error and MUST NOT trigger silent fallback to another destination.
- **FR-013**: Local-only operation MUST remain usable without internet access after installation and any required local model/workspace prerequisites are available.
- **FR-014**: SAM MUST let the user stop, restart, and delete a local workspace and MUST report cleanup failures explicitly.
- **FR-015**: SAM MUST recover saved project and conversation data after an unexpected application exit and MUST report when an active workspace needs recovery.
- **FR-016**: SAM MUST keep saved projects, conversations, and credentials on the user's computer. It MUST send prompt or project content only to the inference provider explicitly selected for that agent or to another remote integration explicitly configured by the user; a saved credential may be used only to authenticate with its associated provider.
- **FR-017**: SAM MUST protect credentials from exposure in user-visible errors and logs, and provide a way to remove saved provider credentials.
- **FR-018**: The local control interface MUST be reachable only from the local computer by default; enabling access from other devices MUST require an explicit user action and clear warning.
- **FR-019**: Upgrade and uninstall flows MUST explain their effect on local data. Uninstall MUST require explicit confirmation before deleting saved data.
- **FR-020**: Local desktop behavior MUST remain separate from the existing hosted deployment; the feature MUST NOT change hosted users' workspace or inference behavior.
- **FR-021**: The first release MUST be limited to Windows and a single local user. Hosted synchronization, multi-user access, remote VM provisioning in local mode, and other desktop operating systems are out of scope.
- **FR-022**: When the user closes SAM while an agent is running, SAM MUST ask whether to stop the work or keep it running in the background. The default MUST be to stop gracefully; continued background work MUST require explicit user choice and remain clearly indicated.
- **FR-023**: Local workspace creation MUST preserve supported repository devcontainer settings, including nested-Docker workflows. Docker Desktop's WSL2 Linux VM remains the security boundary; SAM MUST NOT publish its Docker API beyond that boundary.

### Key Entities

- **Local installation**: The user's desktop copy of SAM, its local services, and lifecycle state.
- **Local project**: A repository or workspace context and its locally saved settings and conversations.
- **Local workspace**: An isolated environment on the user's computer where an agent operates on project files.
- **Agent configuration**: A supported agent's identity, inference mode, selected model/provider, and associated credential reference.
- **Inference connection**: A local model service or configured cloud provider that receives an agent's inference requests.
- **Conversation**: A project's saved user and agent messages, including its associated agent and workspace state.
- **Credential**: A provider secret saved for local use and removable by the user.

### Assumptions

- The first release targets a supported Windows installation and one local user profile.
- Local workspaces require Docker Desktop with WSL2 Linux containers; users install and license Docker Desktop separately.
- OpenCode is the first agent to support local inference; the first verified local service is Ollama using an OpenAI-compatible connection; existing compatible cloud-model options remain available.
- Local inference must not require a SAM-hosted or Cloudflare-hosted service after setup.
- Prompts are sent to a cloud provider only when the user explicitly selects cloud inference for that agent.
- Local data survives application upgrades and restarts. Uninstall offers retention or confirmed deletion rather than deleting user data by default.
- The user can remove stored credentials and delete workspaces independently of project and conversation data.

### Out of Scope

- Hosted synchronization or backup of local projects and conversations.
- Multi-user access to a local installation.
- Provisioning or managing remote VMs from local mode.
- macOS or Linux desktop releases.
- Adding new cloud inference providers beyond the existing supported options.

## Success Criteria

### Measurable Outcomes

- **SC-001**: On a supported Windows machine with Docker Desktop and WSL2 available and no Cloudflare account, a first-time user can launch SAM and create a local project in five minutes or less.
- **SC-002**: In 100% of local-inference verification scenarios, OpenCode uses the selected local model service and no cloud inference provider receives that request.
- **SC-003**: In 100% of cloud-inference verification scenarios, the request reaches the selected cloud provider and no local model service receives that request.
- **SC-004**: In 100% of unavailable-destination scenarios, SAM reports an actionable failure and does not send the request to the other inference mode.
- **SC-005**: In 100% of normal restart verification scenarios, the user's saved projects, agent choices, and completed conversations remain available.
- **SC-006**: In the default configuration, attempts from another device on the local network cannot access the local control interface.
- **SC-007**: Users can remove a saved provider credential and can choose whether to retain or delete local data during uninstall without requiring a hosted account.
- **SC-008**: In the default configuration, the host model service connection provided by SAM cannot be reached from another device on the local network.
- **SC-009**: In 100% of app-close scenarios with active work, SAM asks what to do, defaults to stopping gracefully, and clearly indicates when the user explicitly chooses background operation.
