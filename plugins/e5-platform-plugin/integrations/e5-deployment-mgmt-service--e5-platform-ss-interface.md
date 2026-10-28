---
integration: e5-deployment-mgmt-service <-> e5-platform-ss-interface
protocol: grpc
direction: e5-platform-ss-interface depends on e5-deployment-mgmt-service
---

# e5-deployment-mgmt-service ↔ e5-platform-ss-interface

## What crosses the boundary

The interface's `GrpcChannelConfig` opens one plaintext `ManagedChannel`
(`managementServiceChannel`, host/port from `MANAGEMENT_SERVICE_HOST`/`_PORT`
— defaults `mgmt-service-pl-subsystem:50051`) and uses a single
`DeploymentManagementServiceGrpc.DeploymentManagementServiceBlockingStub`
for every RPC into DMS. For `scaleAppWorker` specifically
(`ScaleWorkerGrpcService` → `DeploymentManagementGrpcClient`):

- `validateScaleAppWorkerVersionRequirement`, `getFbWorkerDetails`,
  `getAllActiveWorkflowManagerRunResponse` — per-app mode only, all keyed on
  `(deploymentVersionId, appName)`.
- `getSharedCredentialGroup(groupName)` — group mode only, added for
  `shared-credential-group` support (see `e5-deployment-mgmt-service`'s
  knowledge file of the same name for the server side). Returns
  `found`/`active`/`total_count`/`common_deployment_count`/`message`. The
  interface currently checks only `found` and `active`; `total_count`/
  `common_deployment_count` are returned but not used by any caller yet.
- `performTask` — records the task and returns a `taskId`, for both modes.

**Proto drift risk:** the interface keeps its own copy of the relevant proto
file (`deployment_mgmt_service.proto`), separate from DMS's
`deployment_management.proto` — not a shared/generated-from-one-source
file. Adding a new RPC on the DMS side means manually mirroring the message
+ rpc definition into the interface's copy; nothing enforces they stay in
sync. Confirmed by checking: as of 2026-10-26, DMS's `SetSharedCredentialGroupActive`
RPC has **no** mirror in the interface's proto at all (the interface never
needed it) — don't assume every DMS RPC is available here just because it
exists server-side.

## Gotcha: `usePlaintext()` either side, no TLS

Both channel builders (`managementServiceChannel`, `workflowOpsServiceChannel`)
use `.usePlaintext()` — fine for local/internal network testing, but worth
knowing before assuming a TLS-related failure is something else.
