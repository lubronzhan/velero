# Restore dry run

## Abstract

Add `velero restore create --dry-run` to preview which resources and volume operations a restore would attempt, using the backup and the current target cluster.
The preview runs on the Velero server and persists logs, results, and a resource list while core restore operations avoid changing application resources or restoring volume data.
Restore plugins are skipped by default; users can explicitly enable plugins that honor the dry-run field.

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

- Preview selected resources and intended create, update, and skip decisions using the core restore rules, with optional plugin execution for plugin-dependent decisions.
- Prevent application-resource writes, hook execution, and volume-data changes in core restore operations, and define the same expectation for explicitly enabled restore plugins.
- Make the preview available through the existing asynchronous restore workflow and artifact download mechanisms.

## Non Goals

- Backup dry run, which can be proposed separately after restore semantics are agreed on.
- Guaranteed success of a subsequent restore, admission validation, capacity checks, or a transactional plan that can later be applied.
- Sandboxing plugins that ignore dry-run mode or rolling back their side effects.

## High-Level Design

A dry run creates a normal Restore custom resource with `spec.dryRun.enabled: true`.
The server downloads the backup and evaluates the restore against the target cluster, using a planning execution mode at every operation that would change application or volume state.
It records intended actions rather than issuing those operations.
The Restore object, its status, preview artifacts in backup storage, and temporary local files are permitted bookkeeping writes.
DownloadRequests and ServerStatusRequests used by the CLI are also permitted.
Core planning operations must not create, patch, update, or delete any application object, namespace, snapshot resource, data-transfer resource, cloud volume, or volume content.
Explicitly enabled restore actions are expected to honor the same restriction, but Velero cannot enforce it inside plugin processes.

Dry runs skip all RestoreItemActions by default, including built-in actions, and clearly identify the preview as incomplete because plugin transformations, additional items, and skip decisions are missing.
Users can opt in with `--run-plugins` once they have verified that their installed restore plugins honor `restore.spec.dryRun.enabled`.
When enabled, actions still transform objects, return additional items, and decide `SkipRestore`, but must avoid external side effects when `restore.spec.dryRun.enabled` is true.
The opt-in is a user assertion about plugin behavior; Velero does not automatically verify plugin support or prevent a plugin that ignores the field from making changes.

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

    // RunPlugins enables RestoreItemActions during a dry run. Users must
    // verify that installed actions honor Enabled before enabling this option.
    // Requires Enabled to be true.
    // +optional
    RunPlugins bool `json:"runPlugins,omitempty"`
}
```

Omitting `dryRun`, using an empty object, or setting `dryRun.enabled: false` preserves normal restore behavior, including execution of applicable restore actions.
Both nested booleans default to false when omitted.
For an enabled dry run, omitted or false `dryRun.runPlugins` skips restore actions; true enables them.
Reject `dryRun.runPlugins: true` unless `dryRun.enabled` is true, including for direct API submissions, using CRD schema validation and controller validation.
Keep dry-run-only settings under this configuration object so they do not appear to control ordinary restores.
Regenerate the Restore CRD and generated API documentation and add matching RestoreBuilder methods.
Treat the execution mode as fixed once processing starts; reject changes to the entire `spec.dryRun` configuration after the initial phase so a preview cannot become a real restore or enable plugins on retry.
Controllers must retain the original execution mode throughout a processing attempt.

Bind a boolean `--dry-run` flag in `pkg/cmd/cli/restore/create.go` and map it to `spec.dryRun.enabled`.
The help text must explain that this submits a server-side preview and persists its results.
Bind a boolean `--run-plugins` flag, default false, and map it to `spec.dryRun.runPlugins`.
Leave the configuration pointer nil for ordinary CLI restores and allocate it when `--dry-run` is enabled.
Reject an explicitly supplied `--run-plugins` flag without `--dry-run` to avoid suggesting that it controls plugins during ordinary restores.
Its help text must state that users should enable it only after verifying that installed restore actions honor `spec.dryRun.enabled`.
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

# Run a new preview after verifying that installed restore actions honor dryRun.
velero restore create preview-2 --from-backup backup-1 \
  --dry-run --run-plugins --wait
```

The corresponding dry-run configuration in a Restore manifest is:

```yaml
spec:
  dryRun:
    enabled: true
    runPlugins: true
```

`restore get` and `restore describe` must visibly identify dry runs, including in structured describe output where applicable.
Also display whether restore plugins were skipped or enabled; when skipped, show that plugin-dependent decisions were not evaluated.
Show counts as resources evaluated, rather than resources restored, when interpreting progress for a dry run.
Creating a real restore requires a new Restore object without `dryRun`; editing or promoting a completed preview is unsupported.

### Selection and planning

Reuse archive traversal, resource priorities, global and fine-grained resource policies, namespace selection, label selectors, namespace mapping, completion checks, and resource modifiers.
When plugins are enabled, also reuse additional-item traversal, deduplication, cycle protection, and existing must-include behavior.
When plugins are skipped, do not infer their transformations or additional items; identify resource decisions as provisional where skipped actions could affect them.
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
A preview with such gaps must not be presented as a fully evaluated restore, whether plugins are enabled or skipped.

### Plugin contract

The Restore already travels in `RestoreItemActionExecuteInput`, so updated plugins can check `input.Restore.Spec.DryRun != nil && input.Restore.Spec.DryRun.Enabled` without changing the Execute payload.
No capability RPC or new plugin interface is required in the initial implementation.
For a dry run with `dryRun.runPlugins` false, bypass restore-action loading, selector resolution, and execution so skipped restore actions do not run setup or AppliesTo code either.
ObjectStore access remains enabled because the server needs to read backups and persist preview artifacts, including when a plugin binary also provides restore actions.

For a dry run with `dryRun.runPlugins` true, resolve and invoke applicable restore actions through the existing plugin workflow.
Before submission, print a notice that plugin execution is enabled and that plugins which ignore `spec.dryRun.enabled` may make real changes.
Do not require an additional confirmation prompt; the explicit flag records the user's choice.
Document the same requirement for direct API clients setting `spec.dryRun.runPlugins`.
Plugin authors should document dry-run support and ensure plugin setup and selector resolution are side-effect free, and that dry-run Execute only performs reads and in-memory transformations.
Plugins must preserve `UpdatedItem`, `AdditionalItems`, and `SkipRestore` semantics where those decisions can be computed without writes.
They must not create resources, call cloud mutation APIs, start data transfer, or return an active OperationID.
An OperationID returned during a dry run is a plugin contract error; Velero must not poll Progress, call Cancel, or wait for additional-item readiness.
The preview records readiness as unchecked rather than asserting that dependencies are ready.

Audit and update all built-in restore actions before enabling the CLI flag, including CSI actions and init-hook injection.
Cloud snapshot restore is bypassed in core planning logic, so VolumeSnapshotter plugins are not invoked to simulate allocating a volume.
ObjectStore plugins remain necessary for reading the backup and writing preview artifacts.
The flag is not a sandbox: plugins remain trusted code with their existing credentials, and enabling an incompatible plugin can violate the no-mutation expectation even if Velero's own planning path performs no application writes.

### Results and lifecycle

Keep the existing Restore phases and status error/warning accounting.
For a dry run, Completed means evaluation in the selected plugin mode completed without errors; it does not mean resources were restored or that a real restore will succeed.
When plugins are skipped, always include a warning in results and logs that transformations, additional items, and plugin skip decisions were not evaluated, even if the preview completes.
This known omission alone does not cause PartiallyFailed; retain the visible plugins-skipped indicator in get/describe output so Completed cannot imply a complete plan.
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

An absent `dryRun` configuration or a configuration with `enabled` false and `runPlugins` false leaves existing restores and plugin behavior unchanged.
The API fields, generated schemas, controller behavior, and CLI must ship together.
Only support previews against servers and CRDs that implement the feature.
An older CRD can prune an unknown field, and an older server can ignore a preserved field, turning an intended preview into a real restore.

Before submitting a dry run, the CLI must check the server version through ServerStatusRequest against the first supporting release and verify that the installed Restore CRD schema exposes an object at `spec.dryRun` with boolean `enabled` and `runPlugins` properties.
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
The initial design instead skips restore actions by default and lets users enable them explicitly after verifying support.
Automatic capability negotiation remains a possible follow-up.

### Always execute restore plugins

Always calling plugins would preserve their planning decisions but allow older plugins that ignore the new field to make real changes without a separate opt-in.
Skipping restore plugins by default makes execution a deliberate choice and makes missing plugin decisions visible in the preview.

## Security Considerations

A dry run uses the server's existing permissions to read the backup and inspect the cluster, and may reveal resource names and configuration decisions to users who can access its artifacts.
Apply existing Restore and DownloadRequest authorization, backup-storage access controls, signed-URL expiry, and log redaction rules.
Do not persist complete resource bodies, Secret values, credentials, or file contents in the preview list or new log messages.
The preview intentionally writes its own bookkeeping and does not establish a new read-only authorization boundary for third-party plugin processes.
With `dryRun.runPlugins` enabled, honoring `dryRun.enabled` is the plugin's responsibility, and Velero cannot guarantee that an incompatible plugin avoids application, cloud, or volume mutations.

## Implementation

Implement this proposal after design review in the following order:

1. Add restore-action bypass and opt-in execution paths, and update built-in actions to honor `spec.dryRun.enabled` while leaving ordinary restores unchanged.
2. Add the Restore API field and generated schemas, and refactor shared restore decisions with explicit planning branches at mutation boundaries.
3. Integrate controller completion, artifact persistence, preview action strings, and finalizer handling.
4. Add the CLI flags, compatibility checks, get/describe output, and user and plugin-author documentation.

The feature is ready only when all mutation paths and the full lifecycle have been verified; exposing the flag before these pieces land would provide an unsafe preview.
No release date or contributor commitment is assumed by this proposal.

Verification must cover:

- Selection parity for core filters, selectors, mapped namespaces, resource policies, and modifiers; with plugins enabled, also verify plugin additional items, must-include items, and duplicate/cyclic dependencies.
- Existing-resource policies, ServiceAccount merging, status restoration, labels, managed fields, generated names, and missing or terminating namespaces with the informer cache enabled and disabled.
- Native snapshots, CSI snapshots and data movement, file-system restore, dynamic provisioning, and in-place restore, including existing PVCs and read-only preflight failures.
- Zero restore-action loading, selector resolution, or execution when plugins are skipped, with warnings and visible mode indicators in persisted and CLI output.
- Opt-in execution for both plugin protocol versions with dry-run-aware actions, SkipRestore, additional items, and erroneous OperationIDs; ordinary restores still execute applicable actions when the dryRun configuration is absent or disabled.
- CLI flag validation and persisted nested dryRun configuration, including omitted and empty objects, direct API submission, rejection of runPlugins without enabled, and rejection of execution-mode changes after processing starts.
- Missing CRDs and unresolved custom resources without CRD establishment or readiness timeouts.
- Hook selection with zero exec calls, zero helper workloads, and zero asynchronous operation polling.
- Full controller and finalizer runs using recording clients that fail unexpected application writes, plus snapshotter and hook fakes that fail unexpected side-effect calls.
- Persisted artifacts, clearly marked terminal states, storage failures, deletion, and failures of old-server/old-schema compatibility checks before Restore submission.
- An end-to-end preview against a populated cluster that compares application objects and volume checksums before and after, including a missing mapped namespace, and verifies that only preview bookkeeping was written.
- Ordinary restore regression coverage and, with dry-run-aware plugins enabled, comparison of planned actions with a real restore in a separate equivalent test cluster, allowing for server-assigned values and unchecked readiness/admission outcomes.

## Open Issues

- Should a future enhancement add automatic plugin capability negotiation or allow opting in to individual restore actions?
- Should planned volume operations eventually have a dedicated structured artifact instead of being recorded only in the restore log?
