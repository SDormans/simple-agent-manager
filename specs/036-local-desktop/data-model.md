# Data Model: Native Local SAM Desktop

## Ownership Boundary

Each desktop installation has one local principal. Local data is owned by that installation and is not synchronized to hosted SAM. Existing hosted users, databases, and workspaces are not migrated or modified by this feature.

Application records live in the app-owned local database. Repository files remain at user-selected paths. Docker workspace resources are tracked by stable SAM installation/workspace labels. Provider secrets live in the Windows user-protected store; database rows contain references, never plaintext secrets.

## Entities

### Local Installation

| Field | Type | Constraints |
|---|---|---|
| `installationId` | opaque ID | Stable for the local data directory; not a user-supplied authorization claim. |
| `schemaVersion` | integer | Advances only after a verified migration and backup. |
| `dataDirectory` | absolute path | Per-user application data path; never inferred from the repository working directory. |
| `runtimeMode` | enum | `local`; selected by the installed desktop runtime, not a request payload. |
| `createdAt` | timestamp | Set once during first-run initialization. |

### Local Principal

| Field | Type | Constraints |
|---|---|---|
| `userId` | opaque ID | One stable local user row for existing ownership relationships. |
| `installationId` | ID | Must match the active installation. |
| `displayName` | string | Local-only and mutable; never used as an identity key. |

The local API authenticates a session for this principal. The renderer does not get to assert its own `userId`.

### Project

| Field | Type | Constraints |
|---|---|---|
| `projectId` | opaque ID | Canonical key; existing project relationships continue to use it. |
| `userId` | ID | Must match the local principal. |
| `name` | string | Display label only. |
| `repositoryPath` | absolute path | User-selected local source; preserve on app-data cleanup. |
| `gitRemote` | optional URL | Optional integration; use only with explicit user configuration and credentials. |
| `createdAt`, `updatedAt` | timestamps | Persisted locally. |

Project path and optional Git remote are distinct: a remote URL is not permission to clone, push, or disclose source without an explicit user action.

### Local Workspace

| Field | Type | Constraints |
|---|---|---|
| `workspaceId` | opaque ID | Stable and distinct from Docker container IDs. |
| `projectId`, `userId` | IDs | Both must match the authorized local project/principal. |
| `runtime` | enum | `docker-desktop-wsl2` for the first release. |
| `containerId` | optional opaque ID | Recorded after successful container creation; Docker labels remain the recovery source. |
| `status` | enum | `stopped`, `starting`, `ready`, `stopping`, `failed`, `cleanup-pending`. |
| `agentSessionId` | optional ID | Canonical link to the agent session for this workspace/chat. |
| `createdAt`, `updatedAt` | timestamps | Used for recovery and diagnostics, not as a substitute for live runtime state. |

State transitions:

```text
stopped -> starting -> ready -> stopping -> stopped
                    \-> failed
ready -> failed
stopping -> cleanup-pending -> stopped
```

`stopped` and `deleted` are reported only after the Docker runtime confirms the effect. Recovery enumerates resources with this SAM installation/workspace's labels; it never prunes unrelated Docker resources.

### Agent Configuration

| Field | Type | Constraints |
|---|---|---|
| `agentType` | string | Must be a supported agent. OpenCode is first for local inference. |
| `inferenceMode` | enum | Explicit `local` or `cloud`; no implicit default when the user has not configured a destination. |
| `provider` | string | Must be compatible with the selected mode and agent. |
| `model` | string | Required when the provider requires an explicit model. |
| `connectionId` | optional ID | References a local model connection or configured cloud credential. |
| `credentialRef` | optional opaque reference | References the OS protected credential store; never a secret value. |
| `updatedAt` | timestamp | Per-agent selection persists across sessions. |

Validation:

- OpenCode local mode requires a verified OpenAI-compatible local connection.
- Cloud mode requires a supported provider and its required credential.
- Provider identity and destination are checked before the agent process starts.
- An unavailable destination is an error; the system never rewrites `inferenceMode`.
- Existing hosted HTTPS validation and provider selection remain unchanged.

### Model Connection

| Field | Type | Constraints |
|---|---|---|
| `connectionId` | opaque ID | Local connection identity, not a URL. |
| `providerType` | enum | `ollama` or `openai-compatible` in the first release. |
| `baseUrl` | URL | Explicitly configured; validated against the local gateway's supported destination policy. Not passed through as an arbitrary agent proxy target. |
| `modelId` | string | Must be present in the service's reported model list or produce a visible verification warning. |
| `credentialRef` | optional opaque reference | For local services that require authentication. Secret stored outside the database. |
| `lastVerifiedAt` | optional timestamp | Informational only; runtime calls still handle connection failure. |

### Conversation and Messages

Reuse the existing project-scoped chat/session and message concepts stored by `ProjectData` Durable Objects where the local runtime proof succeeds. Keep the canonical project/chat/agent-session mapping. Persist streamed agent output to the same conversation it originated from; do not resolve a live session by workspace recency.

### Protected Credential

| Field | Type | Constraints |
|---|---|---|
| `credentialRef` | opaque ID | Database reference only. |
| `provider` | string | Must match the provider expected by the consumer. |
| `secret` | secret-store entry | Windows user-protected storage; never logged or returned to the renderer after save. |
| `createdAt`, `updatedAt` | timestamps | No secret material in diagnostic fields. |

Credential removal deletes the protected entry and its references without deleting project, conversation, or workspace data.

Credential creation and updates must pass through the trusted desktop credential-store boundary. Secret values are never written to the local database, returned to the renderer after saving, injected into workspace settings, or included in logs; only opaque references may cross the local API.

## Cross-Entity Invariants

1. A local API request resolves the authenticated principal server-side; caller-provided IDs do not authorize access.
2. Project, workspace, conversation, and agent-session relationships are checked at each boundary.
3. Local mode may use only local runtime resources; hosted mode remains on the existing provider path.
4. Model gateway tokens are scoped to one workspace and its configured connection, expire with that workspace/session, and cannot select another upstream.
5. Docker operations mutate only resources carrying the matching installation and workspace labels.
6. Local workspace startup preserves supported repository devcontainer settings, including nested-Docker workflows, without moving the trust boundary outside Docker Desktop's WSL2 Linux VM or publishing its Docker API to the local network.
7. Cleanup and migration status reflect observed outcomes rather than requested outcomes.
8. User-selected repository paths and app-owned databases are separate deletion domains.

## Migration and Compatibility

- A new local installation initializes its own database and principal; no hosted user/project data is copied.
- Replay the existing D1 migration chain only if the Worker-runtime feasibility test confirms support. ProjectData Durable Object migrations remain append-only.
- New agent-mode fields must have explicit defaults that preserve existing hosted settings. A local setting must not change hosted behavior when absent.
- Before a local schema migration, create and verify a restorable backup. If backup or migration fails, fail startup without deleting or overwriting the prior state.
