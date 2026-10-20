# Restore dry run

## Abstract

Add `velero restore create --dry-run` to preview which resources and volume operations a restore would attempt, using the backup and the current target cluster.
The preview runs on the Velero server and persists logs, results, and a resource list while core restore operations avoid changing application resources or restoring volume data.
Built-in restore actions always participate in dry runs after being updated to honor dry-run mode; external restore actions are skipped by default and can be explicitly enabled.

## Background

[Issue #448](https://github.com/velero-io/velero/issues/448) requests a way to inspect a partial restore before running it.
Resource and namespace filters, label selectors, namespace mappings, resource modifiers, additional items returned by plugins, and existing resources all influence the result.
Printing the Restore manifest alone does not show which backed-up items these settings select.
The [latest maintainer guidance](https://github.com/velero-io/velero/issues/448#issuecomment-3096799295) asks for a design document before implementation.
This document is a proposal; the commands and API additions below are not implemented yet.

The current restore workflow has side effects before and after its main resource creation call.
For example, `restoreItem` ensures namespaces exist before executing restore actions, `pvRestorer.executePVAction` creates cloud volumes from native snapshots, and the CSI PVC action can create VolumeSnapshots and DataDownloads.
Status updates, ServiceAccount merges, label patches, exec hooks, and restore finalization also require explicit handling.
As a result, adding a conditional check around `resourceClient.Create` would be insufficient.

## Goals

- Preview selected resources and intended create, update, and skip decisions using core restore rules and built-in actions, with optional external action execution.
- Prevent application-resource writes, hook execution, and volume-data changes in core restore operations and built-in actions, and define the same expectation for explicitly enabled external actions; keep dry-run and external-plugin settings fixed once processing starts, including on retries.
- Make the preview available through the existing asynchronous restore workflow and artifact download mechanisms.

## Non Goals

- Backup dry run, which can be proposed separately after restore semantics are agreed on.
- Guaranteed success of a subsequent restore, admission validation, capacity checks, or a transactional plan that can later be applied.
- Sandboxing plugins that ignore dry-run mode or rolling back their side effects.
- Changing dry-run or external-plugin settings after a Restore starts, or converting a preview into a real restore; different settings require a new Restore object.

## High-Level Design

A dry run creates a normal Restore custom resource with `spec.dryRun.enabled: true`.
The server downloads the backup and evaluates the restore against the target cluster, using a planning execution mode at every operation that would change application or volume state.
It records intended actions rather than issuing those operations.
The Restore object, its status, preview artifacts in backup storage, and temporary local files are permitted bookkeeping writes.
DownloadRequests and ServerStatusRequests used by the CLI are also permitted.
Core planning operations and built-in restore actions must not create, patch, update, or delete any application object, namespace, snapshot resource, data-transfer resource, cloud volume, or volume content.
Explicitly enabled external restore actions are expected to honor the same restriction, but Velero cannot enforce it inside external plugin processes.

Dry runs always execute applicable built-in RestoreItemActions, including PVC actions, preserving their transformations, additional items, and `SkipRestore` decisions while bypassing their mutation paths.
Dry runs skip external RestoreItemActions by default and clearly identify any omitted external actions whose transformations, additional items, and skip decisions were not evaluated.
Users can opt in with `--run-external-plugins`, which sets `spec.dryRun.runExternalPlugins: true`, once they have verified that their installed external restore plugins honor `restore.spec.dryRun.enabled`.
When enabled, external actions still transform objects, return additional items, and decide `SkipRestore`, but must avoid external side effects when `restore.spec.dryRun.enabled` is true.
The opt-in is a user assertion about external plugin behavior; Velero does not automatically verify external plugin support or prevent a plugin that ignores the field from making changes.

## Detailed Design

### Restore API and CLI

Add an optional configuration field to `RestoreSpec` and a new `RestoreDryRunConfig` type in `pkg/apis/velero/v1/restore_types.go`:

```go
type RestoreSpec struct {
    // ... existing fields ...

    // DryRun configures restore preview evaluation. Velero still persists
    // the Restore object, its status, logs, results, and resource list.
    // +optional
    DryRun *RestoreDryRunConfig `json:"dryRun,omitempty"`
}

type RestoreDryRunConfig struct {
    // Enabled evaluates planned actions without restoring application
    // resources or volume data through core restore operations.
    // +optional
    Enabled bool `json:"enabled,omitempty"`

    // RunExternalPlugins enables external RestoreItemActions during a dry run.
    // Built-in actions always run. Users must verify that external actions
    // honor Enabled before enabling this option.
    // Requires Enabled to be true.
    // +optional
    RunExternalPlugins bool `json:"runExternalPlugins,omitempty"`
}
```

Omitting `dryRun`, using an empty object, or setting `dryRun.enabled: false` preserves normal restore behavior, including execution of applicable restore actions.
Both nested booleans default to false when omitted.
For an enabled dry run, omitted or false `dryRun.runExternalPlugins` skips external restore actions; true enables them.
Applicable built-in actions execute in both cases and must honor `dryRun.enabled`.
Reject `dryRun.runExternalPlugins: true` unless `dryRun.enabled` is true, including for direct API submissions, using CRD schema validation and controller validation.
Keep dry-run-only settings under this configuration object so they do not appear to control ordinary restores.
Regenerate the Restore CRD and generated API documentation and add matching RestoreBuilder methods.
Treat the execution mode as fixed once processing starts; reject changes to the entire `spec.dryRun` configuration after the initial phase so a preview cannot become a real restore or enable plugins on retry.
Controllers must retain the original execution mode throughout a processing attempt.

Bind a boolean `--dry-run` flag in `pkg/cmd/cli/restore/create.go` and map it to `spec.dryRun.enabled`.
The help text must explain that this submits a server-side preview and persists its results.
Bind a boolean `--run-external-plugins` flag, default false, and map it to `spec.dryRun.runExternalPlugins`.
Leave the configuration pointer nil for ordinary CLI restores and allocate it when `--dry-run` is enabled.
Reject an explicitly supplied `--run-external-plugins` flag without `--dry-run` to avoid suggesting that it controls plugins during ordinary restores.
Its help text must state that it controls external restore actions only and that users should enable it only after verifying that those actions honor `spec.dryRun.enabled`.
Existing `--wait` behavior applies to the preview, and output should identify the request as a dry run.
Existing manifest output flags continue to render a Restore manifest; rendering a manifest does not execute the preview.

Example proposed usage:

```sh
velero restore create preview-1 --from-backup backup-1 \
  --include-namespaces app --namespace-mappings app:app-test \
  --include-resources deployments,services,persistentvolumeclaims \
  --dry-run --wait
velero restore describe preview-1 --details
velero restore logs preview-1

# Run a new preview after verifying that external restore actions honor dryRun.
velero restore create preview-2 --from-backup backup-1 \
  --dry-run --run-external-plugins --wait
```

The corresponding dry-run configuration in a Restore manifest is:

```yaml
spec:
  dryRun:
    enabled: true
    runExternalPlugins: true
```

`restore get` and `restore describe` must visibly identify dry runs, including in structured describe output where applicable.
Also display that built-in actions were enabled and whether external actions were skipped or enabled.
If external actions were omitted, identify their names and show that their decisions were not evaluated; do not label a preview incomplete merely because external execution is disabled when no external restore actions are registered.
Show counts as resources evaluated, rather than resources restored, when interpreting progress for a dry run.
Creating a real restore requires a new Restore object without `dryRun`; editing or promoting a completed preview is unsupported.

### Selection and planning

Reuse archive traversal, resource priorities, global and fine-grained resource policies, namespace selection, label selectors, namespace mapping, completion checks, and resource modifiers.
Always reuse additional-item traversal, deduplication, cycle protection, and existing must-include behavior for built-in actions and any enabled external actions.
When external actions are skipped, do not infer their transformations or additional items; identify resource decisions as provisional where skipped actions could affect them.
Refactor mutation decisions into shared helpers so preview and execution do not acquire separate implementations of existing-resource policy or ServiceAccount merging.
Use read-only cluster queries to choose between create, update, and skip; only a NotFound response means an object is absent.
Authorization, discovery, or transport failures must be reported as failures to evaluate the item, rather than assumed creates.
This must work with both informer-cache settings.

Maintain an in-memory record of planned namespaces and resources, separate from whether an object actually exists in the cluster.
For example, two selected objects in the same missing mapped namespace should produce one planned namespace creation.
Deduplication must not make a planned object appear live to readiness checks or plugins.
Plugins may read the target cluster, but must tolerate dependencies that are only planned and disclose decisions they cannot resolve without those dependencies.

The planning path must handle these operations explicitly:

| Operation | Dry-run behavior |
| --- | --- |
| Ensure target namespace | Read existing namespace state; record a missing namespace as a planned create without creating it or waiting for it. |
| Create resource | Record `would-create` after transformations; do not send a create request. |
| Existing-resource policy | Compare the existing object and the transformed backup object; record `would-update` or `skipped` using the normal policy. |
| ServiceAccount merge | Compute the normal merged object and patch decision in memory; record a planned update when needed. |
| Restore labels, managed fields, and status | Compute intended changes without submitting patches or status updates; describe applicable changes in the item's log entries. |
| Native snapshot restore | Record the snapshot restore intent without calling `CreateVolumeFromSnapshot` or creating a provider volume ID. |
| CSI and file-system restore | Record the intended volume-data restore or dynamic provisioning without creating or updating snapshots, DataDownloads, PodVolumeRestores, or helper workloads. |
| In-place volume restore | Run read-only preflight checks and report their results; never prepare or write the target volume. |
| Restore hooks | Validate and report selected hooks; do not execute exec hooks or create pods that would run init hooks. |
| Readiness and asynchronous operations | Do not wait for planned resources, CRD establishment, volume binding, additional-item readiness, or plugin operations. |
| Restore finalization | Persist preview bookkeeping only; bypass PV patches, snapshot cleanup, volume completion processing, and hook execution/waits. |

Planning volume data must preserve the distinction between restoring a Kubernetes object and restoring its data.
For example, an existing PVC can be skipped while an explicitly requested in-place volume restore is still an intended data operation.
Log the backup volume method, target PVC or PV, intended operation, and reasons for skipping or being unable to evaluate it.
Do not disable `RestorePVs` as a shortcut, since that would change the user's requested restore selection.
Record unavailable generated names and provider volume identifiers as unresolved information instead of inventing values that look like successful allocations.

If a CRD is only present in the backup, it will not become discoverable during a preview.
Use the archived CRD's versions, resource names, kind, and scope to identify dependent custom resources where possible.
If mapping or a plugin decision still depends on resources that would first need to be created, report the item as unevaluated with an explanation.
A preview with such gaps must not be presented as a fully evaluated restore, whether external actions are enabled or skipped.

### Plugin contract

The Restore already travels in `RestoreItemActionExecuteInput`, so updated plugins can check `input.Restore.Spec.DryRun != nil && input.Restore.Spec.DryRun.Enabled` without a separate execution-mode RPC.
No capability RPC or new plugin interface is required in the initial implementation; the additive metadata input described below is needed for built-in CSI data-mover planning.
For every dry run, load, resolve, and execute applicable built-in restore actions in dry-run mode.
Classify actions using server-owned registration provenance and the resolved plugin executable, rather than a plugin's name or `velero.io/` prefix.
Retain the registry's existing rejection of duplicate kind/name registrations; do not introduce external overrides or silently substitute a built-in implementation for a skipped external action.
Preserve normal action ordering and version adaptation for the actions that participate.
For a dry run with `dryRun.runExternalPlugins` false, bypass external restore-action loading, selector resolution, and execution so skipped external actions do not run setup or AppliesTo code either.
Use registry metadata to list omitted external action names without invoking their selectors; the omission warning may conservatively include actions that would not have matched selected resources.
ObjectStore access remains enabled because the server needs to read backups and persist preview artifacts, including when a plugin binary also provides restore actions.

For a dry run with `dryRun.runExternalPlugins` true, resolve and invoke applicable external actions alongside built-in actions through the existing plugin workflow.
Before submission, print a notice that external plugin execution is enabled and that external plugins which ignore `spec.dryRun.enabled` may make real changes.
Do not require an additional confirmation prompt; the explicit flag records the user's choice.
Document the same requirement for direct API clients setting `spec.dryRun.runExternalPlugins`.
Plugin authors should document dry-run support and ensure plugin setup and selector resolution are side-effect free, and that dry-run Execute only performs reads and in-memory transformations.
Plugins must preserve `UpdatedItem`, `AdditionalItems`, and `SkipRestore` semantics where those decisions can be computed without writes.
They must not create resources, call cloud mutation APIs, start data transfer, or return an active OperationID.
An OperationID returned during a dry run is a plugin contract error; Velero must not poll Progress, call Cancel, or wait for additional-item readiness.
The preview records readiness as unchecked rather than asserting that dependencies are ready.

Audit and update all built-in restore actions before enabling the CLI flag, including PVC, CSI, and init-hook injection actions.
Their object transformations and dependency decisions remain active by default, while direct mutations such as snapshot or DataDownload creation become recorded intents.
Cloud snapshot restore is bypassed in core planning logic, so VolumeSnapshotter plugins are not invoked to simulate allocating a volume.
ObjectStore plugins remain necessary for reading the backup and writing preview artifacts.
The flag is not a sandbox: plugins remain trusted code with their existing credentials, and enabling an incompatible plugin can violate the no-mutation expectation even if Velero's own planning path performs no application writes.

### Adapting built-in restore actions

Keeping built-in actions enabled requires preserving their planning decisions while separating those decisions from execution.
Do not return early from every action merely because dry run is enabled: that would lose transformations, dependencies, and `SkipRestore` behavior.
Use a shared nil-safe helper for checking `Restore.Spec.DryRun.Enabled`, and keep ordinary restore execution paths unchanged.

| Built-in action | Existing behavior | Proposed dry-run handling |
| --- | --- | --- |
| Generic PVC (`pkg/restore/actions/pvc_action.go`) | Removes binding annotations, reads PodVolumeBackups, resets binding information, and returns a bound PV as an additional item where appropriate. | Preserve read-only lookups, in-memory transformations, and additional items; the core planner evaluates the returned PV without restoring it. |
| Pod-to-PVC (`add_pvc_from_pod_action.go`) | Returns PVCs referenced by a pod as additional items. | Run normally and traverse the dependencies through the planner. |
| Pod volume restore and init hooks | Read configuration and backup metadata, then inject init containers into the returned pod definition. | Preserve configuration validation and in-memory pod changes; report intended hooks and volume restore steps without creating the pod, PodVolumeRestores, or running containers or exec hooks. |
| CSI PVC without data movement (`actions/csi/pvc_action.go`) | Changes the PVC's data source to the restored snapshot and returns that snapshot as an additional item. | Preserve existing-PVC decisions, deterministic snapshot references, transformations, and additional-item/must-include behavior; plan snapshot dependencies without creating them. |
| DataUpload retrieval (`dataupload_retrieve_action.go`) | Converts an archived DataUpload into DataUploadResult metadata, creates a ConfigMap containing it, and returns SkipRestore. | Compute and validate the same metadata, retain SkipRestore, and supply the metadata through the in-memory planning input described below; do not create a ConfigMap. |
| CSI PVC with data movement | Reads the DataUploadResult ConfigMap, runs preflight checks, may snapshot and delete an existing PVC, creates a DataDownload, and returns an OperationID. | Obtain metadata from planning input, run read-only preflight checks, and record intended operations; do not create snapshots or DataDownloads, delete PVCs, wait for readiness, or return an OperationID. |
| CSI VolumeSnapshot (`actions/csi/volumesnapshot_action.go`) | Renames and transforms the snapshot, returns its content as an additional item, and can create or update a stub VolumeGroupSnapshotContent. | Preserve transformations and dependencies; record stub creation or handle updates as intentions without issuing create, update, or status-update calls. |
| Other built-in transformations | Adjust resource definitions or use read-only configuration/cluster queries. | Preserve those operations after auditing direct calls and helper calls for writes; any mutation requires an explicit planning branch. |

#### Pass DataUpload metadata without a ConfigMap

The current DataUpload retrieval action and CSI PVC data-mover action communicate through a ConfigMap in the Velero namespace.
Skipping the ConfigMap write alone would cause `getDataUploadResult` to fail, so both producer and consumer must change together.
This proposal keeps that ConfigMap out of the permitted preview bookkeeping writes and supplies the required metadata through action input instead.

Extract a shared conversion helper that builds a `v2alpha1.DataUploadResult` from an archived DataUpload and its Backup metadata.
Use the same conversion for ordinary ConfigMap production and dry-run planning, including data mover, snapshot ID, size, snapshot class, source namespace, filesystem, and mover-specific results.
After archive extraction, prepare an in-memory index of these results using original source namespace/PVC identity and the archived DataUpload identity.
Respect the normal DataUpload selection, resource filters, and processing order when making indexed entries available; this preparation must not silently include metadata from a DataUpload the corresponding real restore would exclude.
Resolve the PVC's archived DataUpload reference before namespace mapping and report missing or ambiguous metadata as an item evaluation error.
Do not fall back to an unrelated live ConfigMap or assume an empty volume restore when metadata is unavailable.

Add an optional `DryRunDataUploadResult *v2alpha1.DataUploadResult` field to `RestoreItemActionExecuteInput`.
Populate it only for the built-in CSI PVC action during an enabled dry run, with a copy of the result for that PVC.
Built-in actions run through plugin RPC, so a controller-local map alone is insufficient.
Add a `bytes dryRunDataUploadResult = 5` field containing the JSON-encoded result to Execute requests in both current protocol versions, and thread it through framework clients, servers, version adapters, and restartable wrappers.
This is an additive request field, not a new RPC or a required method on the plugin author interfaces.
Leave it unset for ordinary restores and external actions, and regenerate the protocol bindings.
The CSI PVC action uses this input in dry-run mode and keeps its existing ConfigMap lookup for ordinary restores.
The DataUpload retrieval action still validates its selected archived input and returns SkipRestore during a preview, but performs no ConfigMap write.
The index is scoped to one processing attempt and can be rebuilt from the backup; do not put it in Restore status, annotations, logs, or preview artifacts.

#### Separate CSI planning from volume execution

In the CSI PVC data-mover path, separate metadata lookup and read-only preflight from snapshot creation, PVC deletion, DataDownload submission, cleanup, and operation tracking.
Place the dry-run branch after the checks that can be evaluated from archive metadata and existing resources, but before the first mutation, including deferred cleanup registration.
Return the transformed PVC and log the intended data operation with an empty OperationID.
For incremental in-place restore, report baseline snapshot creation and possible full-restore fallback as conditional intentions; snapshot readiness and changed-block availability cannot be established without execution.
Do not claim that incremental restoration is guaranteed or wait for the baseline snapshot during planning.

In the VolumeSnapshot action, split stub VolumeGroupSnapshotContent calculation from persistence.
Compute the deterministic stub name and required handles in memory, merge planned handles for snapshots sharing a stub, and record whether an existing stub would need an update.
Bypass `CreateVGSC`, `UpdateVGSC`, `UpdateVGSCStatus`, and associated retry or cleanup work while retaining the snapshot/content transformations and dependencies.
Core readiness and finalizer code must also bypass waits and mutations for these planned objects.

#### Verify built-in action behavior

For each action above, compare its returned object, additional items, and SkipRestore decisions between preview and ordinary execution using equivalent input and cluster fixtures, allowing for unresolved execution-generated values.
Use clients that reject all write verbs and fakes that reject snapshot allocation, PVC deletion, data-transfer submission, and hook execution during preview.
Include DataUpload-to-CSI integration across the actual RPC serialization and version adapters, verifying that the consumer receives the correct source PVC metadata with zero ConfigMap writes.
Test namespace mapping, excluded or missing DataUploads, malformed or ambiguous metadata, multiple restores in one plugin process, and ordinary restore regression behavior.
These tests are required before built-in actions can be considered safe to always execute in preview mode.

### Results and lifecycle

Keep the existing Restore phases and status error/warning accounting.
For a dry run, Completed means evaluation in the selected plugin mode completed without errors; it does not mean resources were restored or that a real restore will succeed.
When registered external actions are skipped, include a warning in results and logs naming those actions and explaining that their transformations, additional items, and skip decisions were not evaluated, even if the preview completes.
This known omission alone does not cause PartiallyFailed; retain the visible external-actions-skipped indicator in get/describe output so Completed cannot imply those decisions were evaluated.
Use FailedValidation for invalid input, Failed for fatal evaluation failures, and PartiallyFailed for item evaluation errors, following the existing lifecycle conventions.
Readiness checks and hooks deliberately not executed are documented limitations, while an item that cannot be evaluated is an error.
Keep actual plugin-operation and hook-attempt counters at zero.

Persist the usual restore log, results, and resource-list artifacts through the existing BackupStore methods and DownloadRequest target kinds.
Extend the action strings used by `Request.RestoredResourceList()` with `would-create`, `would-update`, and `unevaluated`; keep `skipped` and `failed` for their existing meanings.
For example, the list can contain `app-test/frontend(would-create)` rather than claiming `app-test/frontend(created)`.
Record mapped target identities and explain source identities, filter decisions, planned secondary updates, hooks, and volume-data operations in the log.
The existing resource-list JSON shape remains `map[string][]string` grouped by API version and kind.
Do not populate restored-volume artifacts with entries that imply completed data transfers; planned volume operations are described in the preview log in this first version.

Do not put a per-item plan in Restore status, since large previews can exceed Kubernetes object size limits.
Large resource lists use the existing compressed object-storage artifact and signed-download workflow.
Artifact publication failures must make the dry run unsuccessful, rather than leaving a successful status with a missing preview.
Upload artifacts before setting a terminal successful phase and preserve diagnostic errors where storage remains available.
Bypass the mutation work in `restore_finalizer_controller.go` for dry runs while retaining the normal status completion bookkeeping.
Deleting a preview may remove its own stored artifacts through the existing external-resources finalizer, but must not clean up application or volume resources.

## Compatibility

An absent `dryRun` configuration or a configuration with `enabled` false and `runExternalPlugins` false leaves existing restores and plugin behavior unchanged.
The API fields, generated schemas, built-in action updates, additive Execute metadata transport, controller behavior, and CLI must ship together.
Only support previews against servers and CRDs that implement the feature.
An older CRD can prune an unknown field, and an older server can ignore a preserved field, turning an intended preview into a real restore.

Before submitting a dry run, the CLI must check the server version through ServerStatusRequest against the first supporting release and verify that the installed Restore CRD schema exposes an object at `spec.dryRun` with boolean `enabled` and `runExternalPlugins` properties.
Abort before creating the Restore if either check fails, times out, or cannot be authorized.
Document the additional read permission needed for this schema check.
Perform these checks only when submitting a dry run, not when rendering a manifest locally.
Direct API clients must perform equivalent checks; manifests with `dryRun` must not be submitted to older installations.
Mixed deployments with old restore controllers and downgrades while previews are pending are unsupported and require draining pending previews before the server change.
Set the concrete minimum server version when the implementation is assigned to a release.

## Alternatives Considered

### Client-side archive inspection

Listing the archive is useful for inspecting backup contents but cannot reproduce server plugin decisions, current-cluster comparisons, or server-configured resource modifiers.
It does not answer which actions a particular restore would attempt.

### Kubernetes API dry-run requests

API-server dry-run writes can validate individual objects but do not simulate a sequence in which namespaces and CRDs become available.
They also do not prevent cloud API changes, plugin side effects, hooks, or volume writes.
This proposal uses read-only planning for application resources; admission validation can be considered separately.

### Require automatic plugin capability negotiation

An optional SupportsDryRun RPC could require explicit support from installed actions before executing them.
This requires framework and plugin updates before previews can use those actions, and a declared capability still depends on the plugin honoring its contract.
The initial design instead updates built-in actions as part of the feature and skips external actions by default, letting users enable external actions explicitly after verifying support.
Automatic capability negotiation remains a possible follow-up.

### Always execute external restore plugins

Always calling plugins would preserve their planning decisions but allow older plugins that ignore the new field to make real changes without a separate opt-in.
Skipping external restore plugins by default makes their execution a deliberate choice and makes missing external plugin decisions visible in the preview.

### Skip built-in actions along with external actions

Skipping built-in actions would omit core transformations and dependencies, including PVC-related decisions, from the default preview.
Built-in actions ship with Velero and can be updated and tested to honor dry-run mode, so they always participate.

## Security Considerations

A dry run uses the server's existing permissions to read the backup and inspect the cluster, and may reveal resource names and configuration decisions to users who can access its artifacts.
Apply existing Restore and DownloadRequest authorization, backup-storage access controls, signed-URL expiry, and log redaction rules.
Do not persist complete resource bodies, Secret values, credentials, or file contents in the preview list or new log messages.
The preview intentionally writes its own bookkeeping and does not establish a new read-only authorization boundary for third-party plugin processes.
With `dryRun.runExternalPlugins` enabled, honoring `dryRun.enabled` is the plugin's responsibility, and Velero cannot guarantee that an incompatible plugin avoids application, cloud, or volume mutations.

## Implementation

Implement this proposal after design review in the following order:

1. Distinguish built-in and external action registrations, add external-action bypass and opt-in execution paths, and adapt built-in actions as described above, including DataUpload metadata conversion and RPC transport.
2. Add the Restore API field and generated schemas, and refactor shared restore decisions with explicit planning branches at mutation boundaries.
3. Integrate controller completion, artifact persistence, preview action strings, and finalizer handling.
4. Add the CLI flags, compatibility checks, get/describe output, and user and plugin-author documentation.

The feature is ready only when all mutation paths and the full lifecycle have been verified; exposing the flag before these pieces land would provide an unsafe preview.
No release date or contributor commitment is assumed by this proposal.

Verification must cover:

- Selection parity for core filters, selectors, mapped namespaces, resource policies, modifiers, and built-in action transformations and dependencies; also verify external action decisions when opted in.
- Additional items, must-include items, and duplicate/cyclic dependencies from built-in actions with external execution disabled and from combined actions when enabled.
- Existing-resource policies, ServiceAccount merging, status restoration, labels, managed fields, generated names, and missing or terminating namespaces with the informer cache enabled and disabled.
- Native snapshots, CSI snapshots and data movement, file-system restore, dynamic provisioning, and in-place restore, including existing PVCs and read-only preflight failures.
- Built-in PVC and CSI action execution with external execution disabled, preserving transformations and dependencies while issuing zero application or volume mutations.
- DataUpload metadata producer/consumer parity, per-PVC Execute input serialization through both protocol versions and adapters, and CSI data-mover planning without ConfigMap creation or lookup.
- Zero external restore-action loading, selector resolution, or execution when external actions are skipped, with omission warnings, action names, and visible mode indicators in persisted and CLI output; no omission warning when no external actions are registered.
- Registration provenance, continued rejection of duplicate built-in kind/name registrations, and external actions using the velero.io prefix, ensuring names alone cannot bypass external opt-in.
- Opt-in execution for both plugin protocol versions with dry-run-aware actions, SkipRestore, additional items, and erroneous OperationIDs; ordinary restores still execute applicable actions when the dryRun configuration is absent or disabled.
- CLI flag validation and persisted nested dryRun configuration, including omitted and empty objects, direct API submission, rejection of runExternalPlugins without enabled, and rejection of execution-mode changes after processing starts.
- Missing CRDs and unresolved custom resources without CRD establishment or readiness timeouts.
- Hook selection with zero exec calls, zero helper workloads, and zero asynchronous operation polling.
- Full controller and finalizer runs using recording clients that fail unexpected application writes, plus snapshotter and hook fakes that fail unexpected side-effect calls.
- Persisted artifacts, clearly marked terminal states, storage failures, deletion, and failures of old-server/old-schema compatibility checks before Restore submission.
- An end-to-end preview against a populated cluster that compares application objects and volume checksums before and after, including a missing mapped namespace, and verifies that only preview bookkeeping was written.
- Ordinary restore regression coverage and, with dry-run-aware plugins enabled, comparison of planned actions with a real restore in a separate equivalent test cluster, allowing for server-assigned values and unchecked readiness/admission outcomes.

## Open Issues

- Should a future enhancement add automatic plugin capability negotiation or allow opting in to individual restore actions?
- Should planned volume operations eventually have a dedicated structured artifact instead of being recorded only in the restore log?
