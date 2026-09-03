---
start_date: 2026-09-02
mlflow_issue: TBD
rfc_pr:
---

# RFC 0014: Cross-workspace MLflow asset copying for fork-and-iterate workflows

| **Date Last Modified** | 2026-09-03 |
| :--------------------- | :--------- |

**Table of contents**

- [Summary](#summary)
- [Basic example](#basic-example)
  - [Fork a prompt](#fork-a-prompt)
  - [Detach a synced prompt](#detach-a-synced-prompt)
  - [Copy or promote a prompt back](#copy-or-promote-a-prompt-back)
- [Motivation](#motivation)
  - [The problem](#the-problem)
  - [Goals](#goals)
  - [User journeys](#user-journeys)
  - [Out of scope](#out-of-scope)
- [Detailed design](#detailed-design)
  - [Terminology and invariants](#terminology-and-invariants)
  - [Core operation modes & semantics](#core-operation-modes--semantics)
  - [Overwrite guardrail and client expectations](#overwrite-guardrail-and-client-expectations)
  - [Asset-specific copy contents](#asset-specific-copy-contents)
  - [Model version provenance tags](#model-version-provenance-tags)
  - [Authorization and visibility](#authorization-and-visibility)
  - [Transactions, conflicts, and idempotency](#transactions-conflicts-and-idempotency)
  - [API](#api)
    - [Typed RPC routes](#typed-rpc-routes)
    - [Copy routes](#copy-routes)
    - [Detach routes](#detach-routes)
    - [Native MlflowClient Python SDK](#native-mlflowclient-python-sdk)
    - [HTTP status and error behavior](#http-status-and-error-behavior)
  - [Database relational schema](#database-relational-schema)
    - [workspace_asset_links](#workspace_asset_links)
    - [workspace_asset_operations](#workspace_asset_operations)
    - [workspace_asset_version_mappings](#workspace_asset_version_mappings)
  - [Read and mutation behavior](#read-and-mutation-behavior)
  - [Rename and delete lifecycle](#rename-and-delete-lifecycle)
- [Acceptance criteria](#acceptance-criteria)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)
- [Open questions](#open-questions)

# Summary

MLflow workspaces support per-asset operations that let users reuse a prompt, registered model, or MLflow MCP registry entry from a source workspace in a target workspace. The operations are strictly metadata-only and designed for fork-and-iterate workflows. The API provides only TWO endpoints per asset type:

- `POST /api/{version}/mlflow/{asset-type}/copy` (supports `SYNC`, `FORK`, and `COPY` modes)
- `POST /api/{version}/mlflow/{asset-type}/detach`

Detailed operation modes:

- **Sync (`relationship_type: "SYNC"`)**: Creates a read-only linked reference via `workspace_asset_links` (`relationship_type: "SYNC"`). When the destination workspace is queried, the MLflow server performs query-time SQL joins against `workspace_asset_links` to resolve source asset metadata in real time. No destination database rows or version mappings are created. Any mutation (`update`, `create_version`, `set_tag`, `delete`) on a synced destination asset is blocked with a read-only error (`INVALID_PARAMETER_VALUE`).
- **Fork (`relationship_type: "FORK"`)**: Creates an independent, editable copy of the asset metadata in destination workspace rows. Artifact locations are preserved as URI pointers (`source`, `storage_location`), with NO artifact byte copying. Creates an association in `workspace_asset_links` (`relationship_type: "FORK"`) for lineage tracking and records source-to-target version mappings in `workspace_asset_version_mappings`. If the target asset name already exists in the destination workspace, the operation fails with `RESOURCE_ALREADY_EXISTS`.
- **Copy / Promote-Back (`relationship_type: "COPY"`)**: Copies an asset to a target workspace, overwriting any same-named asset in the destination. Replaces target metadata and versions atomically if it exists, or creates it if not. Promote-back is executed simply by calling `/copy` with `relationship_type: "COPY"`, overwriting the target asset by name without creating an association link. By design, COPY mode creates NO entry in `workspace_asset_links` so that the copied/promoted asset is completely decoupled from the source. Records the operation in `workspace_asset_operations` with `relationship_type: "COPY"`, attaching immutable version provenance tags (`mlflow.copy.*`).
- **Detach**: Dedicated endpoint `POST /api/{version}/mlflow/{asset-type}/detach` taking `{association_id, target_workspace, idempotency_key}`. Converts an active `SYNC` link into an independent `FORK`, snapshotting current state into local destination database rows and updating `workspace_asset_links.relationship_type = "FORK"`. Subsequent source changes are no longer reflected.
- **Client & UI Guardrail**: Direct MLflow UI modifications for workspace navigation are out of scope for this server-side RFC. Client applications and UI consumers implementing the `COPY` overwrite workflow are expected to display an explicit confirmation dialog before dispatching a request that replaces an existing destination asset.

Unlink operations are out of scope for v1 (stale link cleanup when a source is deleted is addressed in Open Questions / Future Work).

The proposal covers prompts (whose metadata is represented by MLflow registry records), registered models, and MLflow MCP registry entries. It adds typed MLflow RPC-style routes for copy and detach, durable operation records in `workspace_asset_operations`, associations for sync and fork lineage in `workspace_asset_links` (which stores `SYNC` and `FORK` relationships ONLY), and source-to-destination version mappings in `workspace_asset_version_mappings`. It does not copy artifact bytes or manage runtime deployments.

# Basic example

The route identifies the asset type (`prompts`, `registered-models`, or `mcp-servers`). A prompt request cannot select a model or MCP entry in its body. The server checks source and target permissions, executes the transaction, and returns the durable operation result.

## Fork a prompt

The caller sends a `POST` request to `/api/3.0/mlflow/prompts/copy` with `relationship_type: "FORK"`.

```http
POST /api/3.0/mlflow/prompts/copy
Content-Type: application/json

{
  "source_workspace": "shared",
  "source_name": "support-assistant",
  "target_workspace": "team-search",
  "target_name": "support-assistant-team",
  "relationship_type": "FORK",
  "idempotency_key": "op_fork_01J"
}
```

Response:

```json
{
  "resource_type": "prompt",
  "source_workspace": "shared",
  "source_name": "support-assistant",
  "target_workspace": "team-search",
  "target_name": "support-assistant-team",
  "operation_id": "op_01JPROMPTFORK7K4H9M6T2",
  "status": "COMPLETED",
  "copy_mode": "METADATA_ONLY",
  "relationship_type": "FORK",
  "association_id": "op_01JPROMPTFORK7K4H9M6T2",
  "lineage_id": "op_01JPROMPTFORK7K4H9M6T2",
  "source_snapshot_fingerprint": "registry-metadata-sha256:...",
  "creation_time": 1788307200000,
  "last_updated_time": 1788307200000,
  "version_mappings": [
    {"source_version": "1", "target_version": "1"},
    {"source_version": "2", "target_version": "2"}
  ]
}
```

The target rows are editable after this response. If `support-assistant-team` already exists in `team-search`, the request fails with `RESOURCE_ALREADY_EXISTS`.

## Detach a synced prompt

When a prompt is synced, calling `/api/3.0/mlflow/prompts/detach` converts the read-only `SYNC` link into an independent `FORK`.

```http
POST /api/3.0/mlflow/prompts/detach
Content-Type: application/json

{
  "association_id": "assoc_123",
  "target_workspace": "team-search",
  "idempotency_key": "op_detach_456"
}
```

Response:

```json
{
  "relationship": {
    "association_id": "assoc_123",
    "resource_type": "prompt",
    "relationship_type": "FORK",
    "source_workspace": "shared",
    "source_name": "support-assistant",
    "target_workspace": "team-search",
    "target_name": "support-assistant",
    "operation_id": "op_detach_456",
    "lineage_id": "assoc_123",
    "status": "COMPLETED",
    "copy_mode": "METADATA_ONLY",
    "creation_time": 1788307200000,
    "last_updated_time": 1788307250000,
    "source_snapshot_fingerprint": "registry-metadata-sha256:...",
    "version_mappings": [
      {"source_version": "1", "target_version": "1"}
    ]
  }
}
```

## Copy or promote a prompt back

To copy an asset to a target workspace with overwrite semantics (promoting a fork back or copying across workspaces), the canonical invocation uses `/copy` with `relationship_type: "COPY"`. By design, COPY mode creates NO entry in `workspace_asset_links` so that the copied/promoted asset is completely decoupled from the source.

```http
POST /api/3.0/mlflow/prompts/copy
Content-Type: application/json

{
  "source_workspace": "team-search",
  "source_name": "support-assistant-team",
  "target_workspace": "shared",
  "target_name": "support-assistant",
  "relationship_type": "COPY",
  "idempotency_key": "op_promote_789"
}
```

Response:

```json
{
  "resource_type": "prompt",
  "source_workspace": "team-search",
  "source_name": "support-assistant-team",
  "target_workspace": "shared",
  "target_name": "support-assistant",
  "operation_id": "op_01JPROMPTCOPY9L2K8N",
  "status": "COMPLETED",
  "copy_mode": "METADATA_ONLY",
  "relationship_type": "COPY",
  "association_id": null,
  "lineage_id": "op_01JPROMPTCOPY9L2K8N",
  "source_snapshot_fingerprint": "registry-metadata-sha256:...",
  "creation_time": 1788307200000,
  "last_updated_time": 1788307300000,
  "version_mappings": [
    {"source_version": "1", "target_version": "1"},
    {"source_version": "2", "target_version": "2"}
  ]
}
```

The operation overwrites target metadata and versions atomically if `support-assistant` exists in `shared` workspace, or creates it if absent. No entry is stored in `workspace_asset_links`.

# Motivation

## The problem

Workspace isolation provides security boundary and access control, but creates friction when iterating across team boundaries. A prompt, registered model, or MCP server published in a central shared workspace often needs to be reused or adapted in a team workspace without altering the original asset. Conversely, when a team improves an asset, promoting those changes back to the parent workspace should be seamless and unambiguous.

Today, treating these operations as ad hoc create and update calls leaves critical gaps:
- Unclear semantics on whether a destination asset is a live view, a tracked fork, or a decoupled overwrite copy.
- Inconsistent copying of versions, tags, aliases, and URI metadata.
- Risk of accidental target overwrite without guardrails or explicit user warnings.
- Lack of durable operational audit records and source-to-target version lineage.

MLflow's cross-workspace copying feature addresses these gaps by establishing a formal contract and unified database storage layer for asset `SYNC`, `FORK`, `COPY`, and `DETACH` operations.

## Goals

- Provide unified `SYNC`, `FORK`, and `COPY` operation modes on the `/copy` endpoint, alongside a dedicated `/detach` route across prompts, registered models, and MCP servers.
- Keep `SYNC` assets read-only and resolved dynamically via query-time SQL joins without creating destination database rows.
- Ensure `FORK` copies are independent, editable, and retain traceable lineage via `workspace_asset_links` (`relationship_type: "FORK"`), returning `RESOURCE_ALREADY_EXISTS` if target already exists.
- Ensure `COPY` mode replaces target metadata and versions atomically, creating **NO entry in `workspace_asset_links`** so assets are completely decoupled.
- Support promote-back workflows via `/copy` with `relationship_type: "COPY"`, overwriting the target asset by name without creating an association link.
- Support client-side overwrite guardrails for `COPY` mode operations.
- Preserve immutable provenance tags on model versions and enforce protection against modification or deletion.
- Verify source read permission and target write permission using native MLflow authorization primitives.
- Ensure safe retries via `idempotency_key`.

## User journeys

1. **Curated prompt synced for live discovery**: A user syncs a prompt from `shared` workspace into a personal workspace (`relationship_type: "SYNC"`). MLflow creates a link record in `workspace_asset_links`. Reads in the target workspace perform query-time SQL joins to resolve source metadata live. Destination rows are not created, and any attempt to mutate the synced asset returns a read-only error (`INVALID_PARAMETER_VALUE`).
2. **Synced prompt detached into independent fork**: A user decides to customize a synced prompt and calls `POST /api/3.0/mlflow/prompts/detach`. MLflow snapshots the current source prompt state, materializes destination database rows and version mappings, and updates `workspace_asset_links.relationship_type` to `"FORK"`. The user can now edit the target asset independently.
3. **Curated model directly forked to team workspace**: A user forks a registered model using `POST /api/2.0/mlflow/registered-models/copy` with `relationship_type: "FORK"`. Full model metadata, versions, tags, and aliases are materialized in the target workspace. If target asset exists, it returns `RESOURCE_ALREADY_EXISTS`. Version `source` and `storage_location` URI pointers are preserved as metadata; zero artifact bytes are transferred.
4. **Team asset promoted back / overwrite copied to target workspace**: After refining a prompt or model fork, a user invokes `POST /api/{version}/mlflow/{asset-type}/copy` with `relationship_type: "COPY"`. The client application presents an explicit overwrite warning confirmation before proceeding. MLflow validates permissions, atomically replaces target asset metadata and versions, records the operation in `workspace_asset_operations` (`relationship_type: "COPY"`), and creates **NO row in `workspace_asset_links`**, leaving assets completely decoupled.
5. **MCP server configuration copied**: A user copies an MCP server entry via `POST /api/2.0/mlflow/mcp-servers/copy`. Server configuration, versions, tags, and endpoints are copied as metadata, decoupled from runtime container lifecycle.

## Out of scope

- Single-version copying or partial version selection (copying applies strictly to the complete registered model asset).
- Experiments, run tracking data, and run artifacts (deferred; operations focus strictly on AI asset registries).
- Evaluation datasets.
- MLflow traces and trace data.
- Bulk workspace-level migrations (operations are strictly per-asset).
- Bi-directional real-time synchronization (synchronization is strictly one-way, source-to-destination).
- Frontend UI controls for cross-workspace browsing and operations (delegated to external client applications and platform integrations).
- Platform-specific container orchestration or infrastructure lifecycle management.
- Artifact byte copying, checksumming, or storage repository transfers.
- Unlink operation in v1 (stale link cleanup when a source is deleted is addressed in Open Questions / Future Work).
- Asynchronous background/saga execution models.

# Detailed design

## Terminology and invariants

- **Asset Identity**: Uniquely identified by `(workspace, name)` for resource types `prompt`, `registered_model`, and `mcp_server`.
- **`workspace_asset_links`**: The association table recording active `SYNC` and `FORK` lineage. Stores `SYNC` and `FORK` relationships ONLY; `COPY` mode creates no rows here.
- **`workspace_asset_operations`**: The durable audit log and idempotency operations table recording all operations (`SYNC`, `FORK`, `COPY`, `detach`).
- **`workspace_asset_version_mappings`**: The table mapping source version identifiers to target version identifiers.

Key invariants:
1. `SYNC` creates a row in `workspace_asset_links` with `relationship_type = "SYNC"`. It creates no destination asset rows and no version mappings. Queries in the target workspace perform SQL joins against `workspace_asset_links` to resolve source content live.
2. Synced assets are read-only. Any attempt to modify, set tags, create versions, or delete a synced asset returns `INVALID_PARAMETER_VALUE`.
3. `FORK` creates destination rows, records source-to-target version mappings in `workspace_asset_version_mappings`, and maintains an association in `workspace_asset_links` (`relationship_type = "FORK"`). If target asset already exists in the target workspace, `FORK` fails with `RESOURCE_ALREADY_EXISTS`.
4. `COPY` copies an asset to a target workspace, overwriting any same-named asset in destination metadata and versions atomically (or creating it if absent). By design, COPY mode creates NO entry in `workspace_asset_links` so that the copied/promoted asset is completely decoupled from the source. Records operation in `workspace_asset_operations` (`relationship_type = "COPY"`) and attaches immutable version provenance tags (`mlflow.copy.*`).
5. `DETACH` converts an active `SYNC` association to `FORK`, snapshots current source state, materializes destination rows, records version mappings, and updates `workspace_asset_links.relationship_type = "FORK"`.

## Core operation modes & semantics

| Mode / Route | Destination Asset Rows | Read Behavior | Destination Mutations | Association Table (`workspace_asset_links`) | Operations Audit Log (`workspace_asset_operations`) | Version Mappings |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Sync** (`/copy` with `relationship_type: "SYNC"`) | None | Resolves source state in real time via query-time SQL joins | Blocked with `INVALID_PARAMETER_VALUE` error | Active row (`relationship_type: "SYNC"`) | Recorded (`relationship_type: "SYNC"`) | None |
| **Fork** (`/copy` with `relationship_type: "FORK"`) | Materialized in target workspace | Independent target asset rows | Allowed | Active row (`relationship_type: "FORK"`). Fails with `RESOURCE_ALREADY_EXISTS` if target exists | Recorded (`relationship_type: "FORK"`) | Recorded in `workspace_asset_version_mappings` |
| **Copy / Promote-Back** (`/copy` with `relationship_type: "COPY"`) | Replaces/creates target asset rows atomically | Independent target asset rows | Allowed | **No entry created** (assets are completely decoupled) | Recorded (`relationship_type: "COPY"`) | Recorded in `workspace_asset_version_mappings` |
| **Detach** (`/detach`) | Materialized in target workspace | Independent target asset rows | Allowed | Updated from `SYNC` to `relationship_type: "FORK"` | Recorded (`relationship_type: "FORK"`, operation: `detach`) | Recorded in `workspace_asset_version_mappings` |

## Overwrite guardrail and client expectations

- **Client Scope**: Cross-workspace copy, sync, and detach operations are exposed via server REST APIs and the native Python `MlflowClient`. Direct MLflow UI enhancements for multi-workspace navigation are out of scope.
- **Overwrite Confirmation Guardrail**: For `COPY` mode operations (which perform atomic replacement of target asset metadata and versions in the destination workspace), client applications and web consoles are expected to display an explicit confirmation dialog before dispatching the API request to prevent accidental data loss.

## Asset-specific copy contents

| Asset Type | Copied Metadata | Excluded / System Managed |
| :--- | :--- | :--- |
| **Prompt** | Prompt template text, descriptions, version history, asset/version tags, and aliases. (Prompts map physically to registered_model schema). | Workspace ownership, internal database primary keys, permissions. |
| **Registered Model** | Model description, asset tags, version history, version tags, aliases, and exact `source`/`storage_location` URI strings. | Workspace ownership, internal primary keys, permissions. Artifact bytes are never copied. |
| **MCP Server** | Server configuration, description, version history, tags, and endpoints. | Secret values and runtime container deployment state. |

### Storage URI pointer semantics and runtime access prerequisites

- Model versions store artifact locations as URI pointers (`source`, `storage_location`), which are preserved verbatim without transferring binary bytes.
- Downstream execution environments (such as model serving, batch inference, or evaluation runtimes) are expected to hold read credentials for the referenced storage URI scheme (e.g., shared object storage bucket access, cross-account IAM permissions, or container registry credentials).
- Physical binary data replication across air-gapped or isolated storage repositories is an out-of-band operational concern (handled by dedicated storage replication pipelines), decoupled from MLflow's metadata catalog operations.

## Model version provenance tags

When model versions are copied during a `FORK`, `DETACH`, or `COPY` operation, MLflow automatically attaches immutable provenance tags to each target `ModelVersion` entity:

- `mlflow.copy.operation_id`: ID of the copy operation recorded in `workspace_asset_operations`.
- `mlflow.copy.source_workspace`: Name of the source workspace.
- `mlflow.copy.source_name`: Name of the source registered model.
- `mlflow.copy.source_version`: Source version string.
- `mlflow.copy.source_uri`: Recorded source URI pointer metadata (stored as string metadata without dereferencing or accessing artifact repos).

To maintain audit integrity, `set_model_version_tag` and `delete_model_version_tag` in the MLflow tracking store validate and reject any attempt to modify or delete these `mlflow.copy.*` tags (`_validate_copy_provenance_tag_mutation`).

## Authorization and visibility

Authorization uses native MLflow permissions primitives:

- **Source Read Permission**: The requesting user must have `can_read` (or workspace read capability) on the source workspace and asset.
- **Target Write Permission**: The requesting user must have `can_update` or `can_use` (or workspace write capability) on the target workspace.

For overwrite `copy` (including promote-back), the user must have read access on the source asset and `can_update` / write permission on the target destination workspace asset.

Permissions checks are evaluated before any database mutations are performed.

## Transactions, conflicts, and idempotency

All copy and detach operations run within a single SQL transaction:

1. Validate input parameters and permissions.
2. Read source snapshot or validate association state.
3. Check target conflicts:
   - For `FORK` mode: Check if target asset already exists in target workspace. If so, abort transaction and return `RESOURCE_ALREADY_EXISTS` (`409 Conflict`).
   - For `COPY` mode: Check if target asset exists. If so, atomically replace target metadata and versions; if not, create target asset.
4. Perform database mutations (insert target asset rows, insert version mappings, insert/update link row for SYNC/FORK, insert operation record).
5. Commit the transaction atomically.

Idempotency is supported via the `idempotency_key` field (accepted in JSON request body or `Idempotency-Key` HTTP header). `workspace_asset_operations` enforces a scoped unique constraint on `(target_workspace, resource_type, created_by, idempotency_key)`, where `created_by` defaults to an empty string `''` when unauthenticated. This prevents cross-asset key collisions and ensures deterministic uniqueness across all SQL backends. Retrying a request with the same idempotency key returns the existing operation result.

# API

## Typed RPC routes

Cross-workspace operations use typed RPC endpoints under `/api/2.0/mlflow/` and `/api/3.0/mlflow/`:

- **Copy Routes**:
  - `POST /api/3.0/mlflow/prompts/copy`
  - `POST /api/2.0/mlflow/registered-models/copy`
  - `POST /api/2.0/mlflow/mcp-servers/copy`
- **Detach Routes**:
  - `POST /api/3.0/mlflow/prompts/detach`
  - `POST /api/2.0/mlflow/registered-models/detach`
  - `POST /api/2.0/mlflow/mcp-servers/detach`

## Copy routes

Request Body (`CopyPrompt`, `CopyRegisteredModel`, `CopyMCPServer`):

```json
{
  "source_workspace": "shared",
  "source_name": "support-assistant",
  "target_workspace": "team-search",
  "target_name": "support-assistant-team",
  "relationship_type": "FORK",
  "idempotency_key": "optional-key"
}
```

Field rules:
- `source_workspace` (required): Source workspace identifier.
- `source_name` (required): Source asset name.
- `target_workspace` (required): Destination workspace identifier.
- `target_name` (optional): Destination asset name (defaults to `source_name`).
- `relationship_type` (optional): Mode of copy operation:
  - `"SYNC"`: Creates read-only linked reference in `workspace_asset_links`.
  - `"FORK"` (default): Materializes independent copy in target workspace and records association in `workspace_asset_links`. Fails with `RESOURCE_ALREADY_EXISTS` if target asset exists.
  - `"COPY"`: Overwrites same-named asset in destination workspace (or creates if absent). Creates NO entry in `workspace_asset_links` so assets remain completely decoupled.
- `idempotency_key` (optional): Key for request deduplication. Can also be supplied via `Idempotency-Key` HTTP header.

Response Body:

```json
{
  "resource_type": "prompt",
  "source_workspace": "shared",
  "source_name": "support-assistant",
  "target_workspace": "team-search",
  "target_name": "support-assistant-team",
  "operation_id": "op_01J...",
  "status": "COMPLETED",
  "copy_mode": "METADATA_ONLY",
  "relationship_type": "FORK",
  "association_id": "op_01J...",
  "lineage_id": "op_01J...",
  "source_snapshot_fingerprint": "...",
  "creation_time": 1788307200000,
  "last_updated_time": 1788307200000,
  "version_mappings": [
    {"source_version": "1", "target_version": "1"},
    {"source_version": "2", "target_version": "2"}
  ]
}
```

*Note*: For `relationship_type: "COPY"`, `association_id` in the response is `null` because no row is inserted into `workspace_asset_links`.

## Detach routes

Request Body (`DetachPrompt`, `DetachRegisteredModel`, `DetachMCPServer`):

```json
{
  "association_id": "assoc_123",
  "target_workspace": "team-search",
  "idempotency_key": "optional-key"
}
```

Response Body:

```json
{
  "relationship": {
    "association_id": "assoc_123",
    "resource_type": "prompt",
    "relationship_type": "FORK",
    "source_workspace": "shared",
    "source_name": "support-assistant",
    "target_workspace": "team-search",
    "target_name": "support-assistant",
    "operation_id": "op_detach_456",
    "lineage_id": "assoc_123",
    "status": "COMPLETED",
    "copy_mode": "METADATA_ONLY",
    "creation_time": 1788307200000,
    "last_updated_time": 1788307250000,
    "source_snapshot_fingerprint": "...",
    "version_mappings": [
      {"source_version": "1", "target_version": "1"}
    ]
  }
}
```

## Native MlflowClient Python SDK

The Python SDK exposes 6 native methods on `mlflow.client.MlflowClient`:

```python
# Prompt operations
client.copy_prompt(
    source_workspace, source_name, target_workspace,
    target_name=None, idempotency_key=None, relationship_type=None
)
client.detach_prompt(association_id, target_workspace, idempotency_key=None)

# Registered model operations
client.copy_registered_model(
    source_workspace, source_name, target_workspace,
    target_name=None, idempotency_key=None, relationship_type=None
)
client.detach_registered_model(association_id, target_workspace, idempotency_key=None)

# MCP server operations
client.copy_mcp_server(
    source_workspace, source_name, target_workspace,
    target_name=None, idempotency_key=None, relationship_type=None
)
client.detach_mcp_server(association_id, target_workspace, idempotency_key=None)
```

In `copy_*` methods, `relationship_type` accepts `"SYNC"`, `"FORK"`, or `"COPY"` (default `"FORK"`).

## HTTP status and error behavior

| Status | Condition | Meaning |
| :--- | :--- | :--- |
| `200 OK` / `201 Created` | Request completed successfully | Operation committed durable records |
| `400 Bad Request` | Missing required fields, invalid parameters, or mutation on read-only SYNC asset | Returns `INVALID_PARAMETER_VALUE` error |
| `403 Forbidden` | Authorization failure | Insufficient permissions on source or target workspace |
| `404 Not Found` | Source asset or association ID not found | Returns `RESOURCE_DOES_NOT_EXIST` error |
| `409 Conflict` | Target asset name conflict on `FORK` | Returns `RESOURCE_ALREADY_EXISTS` error when target asset already exists in target workspace during a `FORK` operation |

# Database relational schema

The cross-workspace asset copy feature uses a unified database schema with three tables prefixed by `workspace_asset_*`:

## `workspace_asset_links`

Stores active `SYNC` and `FORK` associations ONLY. Mode `COPY` operations create NO entries in this table, ensuring parent and copied assets remain completely decoupled.

```sql
CREATE TABLE workspace_asset_links (
    association_id VARCHAR(32) NOT NULL,
    resource_type VARCHAR(64) NOT NULL,
    relationship_type VARCHAR(16) NOT NULL, -- "SYNC" or "FORK" ONLY
    source_workspace VARCHAR(63) NOT NULL,
    source_name VARCHAR(256) NOT NULL,
    target_workspace VARCHAR(63) NOT NULL,
    target_name VARCHAR(256) NOT NULL,
    operation_id VARCHAR(32) NOT NULL,
    lineage_id VARCHAR(32) NOT NULL,
    created_by VARCHAR(256),
    creation_time BIGINT NOT NULL,
    last_updated_time BIGINT NOT NULL,
    source_snapshot_fingerprint VARCHAR(64),
    CONSTRAINT workspace_asset_links_pk PRIMARY KEY (association_id),
    CONSTRAINT workspace_asset_links_target_uk UNIQUE (target_workspace, target_name, resource_type)
);

CREATE INDEX idx_workspace_asset_links_source ON workspace_asset_links (source_workspace, source_name);
CREATE INDEX idx_workspace_asset_links_target ON workspace_asset_links (target_workspace, target_name);
```

## `workspace_asset_operations`

Durable audit log and idempotency table for all cross-workspace operations (SYNC, FORK, COPY, detach). For `COPY` mode operations, `relationship_type` is `"COPY"` and `association_id` is NULL.

```sql
CREATE TABLE workspace_asset_operations (
    operation_id VARCHAR(32) NOT NULL,
    idempotency_key VARCHAR(256),
    created_by VARCHAR(256) NOT NULL DEFAULT '',
    resource_type VARCHAR(64) NOT NULL,
    source_workspace VARCHAR(63) NOT NULL,
    source_name VARCHAR(256) NOT NULL,
    target_workspace VARCHAR(63) NOT NULL,
    target_name VARCHAR(256) NOT NULL,
    status VARCHAR(32) NOT NULL,
    version_mappings TEXT NOT NULL,
    source_version VARCHAR(256),
    creation_time BIGINT NOT NULL,
    request_fingerprint VARCHAR(64),
    copy_mode VARCHAR(64),
    error_code VARCHAR(64),
    error_message TEXT,
    last_updated_time BIGINT,
    relationship_type VARCHAR(16), -- "SYNC", "FORK", or "COPY"
    association_id VARCHAR(32),   -- NULL for COPY mode operations
    lineage_id VARCHAR(32),
    source_snapshot_fingerprint VARCHAR(64),
    CONSTRAINT workspace_asset_operations_pk PRIMARY KEY (operation_id),
    CONSTRAINT workspace_asset_operations_idempotency_uk UNIQUE (target_workspace, resource_type, created_by, idempotency_key)
);

CREATE INDEX idx_workspace_asset_operations_source ON workspace_asset_operations (source_workspace, source_name);
CREATE INDEX idx_workspace_asset_operations_target ON workspace_asset_operations (target_workspace, target_name);
```

## `workspace_asset_version_mappings`

Stores source-to-target version mappings for materialized copies (`FORK`, `COPY`, `DETACH`).

```sql
CREATE TABLE workspace_asset_version_mappings (
    operation_id VARCHAR(32) NOT NULL,
    source_workspace VARCHAR(63) NOT NULL,
    source_name VARCHAR(256) NOT NULL,
    source_version VARCHAR(256) NOT NULL,
    target_workspace VARCHAR(63) NOT NULL,
    target_name VARCHAR(256) NOT NULL,
    target_version VARCHAR(256) NOT NULL,
    created_by VARCHAR(256),
    creation_time BIGINT NOT NULL,
    CONSTRAINT workspace_asset_version_mappings_pk PRIMARY KEY (operation_id, source_workspace, source_name, source_version),
    CONSTRAINT workspace_asset_version_mappings_uk UNIQUE (operation_id, target_workspace, target_name, target_version)
);

CREATE INDEX idx_workspace_asset_version_mappings_target ON workspace_asset_version_mappings (target_workspace, target_name, target_version);
```

# Read and mutation behavior

- **Read Path for `SYNC`**: When a workspace lists or fetches assets, MLflow performs query-time SQL joins against `workspace_asset_links` where `relationship_type = "SYNC"`. The returned asset representations combine the target identity in the requested workspace with live source metadata.
- **Mutation Path for `SYNC`**: Any write or update API called on a synced asset in the target workspace (such as `update_registered_model`, `create_model_version`, `set_registered_model_tag`, `delete_registered_model`) checks `workspace_asset_links`. If a `SYNC` association exists for that target, the operation is rejected immediately with an `INVALID_PARAMETER_VALUE` error stating that synced assets are read-only.
- **Read/Mutation Path for `FORK` and `COPY`**: Forked and copied assets exist as standard rows in target database tables. They are fully editable independently of the source.

### Query-time SQL pagination: Polymorphic UNION ALL Virtual Subqueries

For search and list operations (such as `search_registered_models` and `search_model_versions`), MLflow avoids application-layer memory stitching ($O(N)$ heap loading, sorting, and slicing). Instead, MLflow pushes filtering, sorting, and pagination down directly to the database engine using an ANSI SQL `UNION ALL` subquery structure:

- **Component 1 (Native rows)**: Selects native asset rows in the queried workspace (`workspace = :target_workspace`).
- **Component 2 (Synced virtual rows)**: Joins `workspace_asset_links` (where `relationship_type = 'SYNC'`) with source asset tables, projecting and aliasing `target_workspace AS workspace` and `target_name AS name` while selecting the underlying source asset metadata.
- **Component 3 (Push-down of Filtering, Ordering, and Pagination)**: The outer query wraps the combined `UNION ALL` subquery, applying client filter clauses (`WHERE`), composite sorting (`ORDER BY`), and cursor pagination (`LIMIT :max_results + 1 OFFSET :offset`) directly at the database engine level.

This design guarantees:
- **100% ANSI SQL standard compliance** across all four supported relational backends (PostgreSQL, MySQL, SQLite, MSSQL).
- **$O(\text{limit})$ memory efficiency**, avoiding full-table scans or memory spikes on workspaces with many synced assets.
- **Deterministic cursor pagination** across page iterations.

# Rename and delete lifecycle

- **Rename Source or Target**: Renaming an asset via `rename_registered_model` cascades atomically to `workspace_asset_links` within the exact same database transaction. If a source asset is renamed, active `SYNC` links update `source_name` to follow the renamed asset, preventing broken links or 404 errors. If a target asset is renamed in its destination workspace, `target_name` is updated.
- **Delete Source**: Deleting a source asset renders active `SYNC` links pointing to it in an orphaned state (source unavailable), with unlinking or detach options available to callers. Forked and copied assets remain completely unaffected since their target rows and URI pointers are materialized locally.
- **Delete Fork or Copy Target**: Deleting a target asset removes the target rows and cleans up any associated link.

# Acceptance criteria

1. **Typed `copy` and `detach` Endpoints**: RPC endpoints and Python SDK support `copy` (with `SYNC`, `FORK`, and `COPY` modes) and `detach` routes for prompts, registered models, and MCP servers.
2. **Decoupled `COPY` Mode**: `COPY` mode replaces target metadata and versions atomically if target exists (or creates it if absent) and creates **NO entry in `workspace_asset_links`**, leaving assets completely decoupled.
3. **Target Conflict Error on `FORK`**: `FORK` mode checks target asset existence and raises `RESOURCE_ALREADY_EXISTS` (`409 Conflict`) if the target asset already exists in the target workspace.
4. **Query-time `SYNC` Resolution**: `SYNC` links resolve live source metadata via SQL joins at query time without creating target asset rows or version mappings.
5. **Read-Only `SYNC` Enforcement**: Any attempt to mutate a synced target asset returns an `INVALID_PARAMETER_VALUE` error.
6. **Dedicated `DETACH` Endpoint**: Converts active `SYNC` link to `FORK`, snapshotting current state into target database rows and updating `workspace_asset_links.relationship_type = "FORK"`.
7. **Client Overwrite Guardrail Guidance**: Client applications invoking `COPY` mode against an existing target asset are provided with clear documentation recommending a client-side confirmation step before executing destination overwrites.
8. **Immutable Model Version Provenance Tags**: Copied model versions receive immutable `mlflow.copy.*` tags that cannot be modified or deleted via tag mutation endpoints.
9. **Permissions & Idempotency**: Source `can_read` and target `can_update` permissions evaluated using native MLflow authorization primitives; safe retries guaranteed via `idempotency_key`.

# Drawbacks

- Query-time joins for `SYNC` assets are pushed down to SQL via `UNION ALL` subqueries; while application memory overhead is prevented, query planning complexity increases slightly on the database engine.
- Promoting back replaces parent asset metadata and version history with the fork's content rather than merging version trees.

# Alternatives

- **Artifact byte copying**: Copying underlying model artifacts alongside metadata. Rejected because synchronously transferring multi-gigabyte or hundred-gigabyte model weights through the MLflow tracking server would saturate server CPU, heap memory, and network throughput, leading to HTTP connection timeouts and requiring complex asynchronous saga orchestration. Preserving URI pointers provides an instantaneous, zero-byte copy operation that leaves binary management to underlying storage layers.
- **Separate per-asset table prefixes**: Using `prompt_links`, `model_links`, etc. Rejected in favor of the unified `workspace_asset_*` table schema.

# Adoption strategy

1. Apply Alembic migrations to create `workspace_asset_links`, `workspace_asset_operations`, and `workspace_asset_version_mappings`.
2. Expose typed REST handlers and `MlflowClient` SDK methods.
3. Expose the operations through the Python `MlflowClient` SDK and REST APIs for adoption by external platforms and client applications.

# Open questions

- Retention policies for historical operation records in `workspace_asset_operations`.
- Recovery and cleanup user experience for orphaned `SYNC` links when a source asset is deleted.
