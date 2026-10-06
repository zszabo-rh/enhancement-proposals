# Testplan — OSAC-5813

## Overview

- Feature: [OSAC-5813](https://redhat.atlassian.net/browse/OSAC-5813), NetApp tenant onboarding/offboarding.
- Design: [design.md](design.md), proposed contracts IC-1–IC-6.
- Total: 12 cases; 5/5 PRD source anchors and 6/6 interface changes mapped. R1–R5 are local references to existing unnumbered PRD text, defined in [design §5](design.md#5-interface-changes), not new requirements.
- Status: planned, not executed. Automated classifications describe intended tests. Live ONTAP/AAP/FC execution and native-path agreement remain prerequisites.

## Planning evidence / execution paths

This matrix follows [Integration testing](https://github.com/osac-project/osac/blob/c8d0d8890dc0e381f70a43ab8572c6b03b429565/docs/INTEGRATION-TESTING.md). DEV owns Unit, Envtest, component integration and Contract; QE owns deployed E2E. Existing fixture tests do not prove a live NetApp boundary.

| Component / behavior / refs | Tier / owner / cases | Suite and execution | Real dependencies / doubles / gaps |
|---|---|---|---|
| Fulfillment ONTAP validation/probe, R1/R2, IC-1/2 | Unit / configuration DEV; TC-R1-01, TC-R2-01 | Extend `fulfillment-service/internal/servers/private_storage_{backends,tiers}_server_test.go`; `ginkgo run internal/servers` from fulfillment-service | Real handlers/validation; local HTTPS ONTAP double for probe; not a real array. |
| CLI → API → persistence, R1, IC-1/3 | Component integration / configuration DEV; TC-R1-02 | Proposed `fulfillment-service/it/it_netapp_storage_configuration_test.go`; existing command `make -C ../osac-installer test PLATFORM=kind PROFILE=dev NS=osac SUITE=fulfillment` from fulfillment-service | Real CLI/service/PostgreSQL; proposed reachable HTTPS probe double; requires installer Kind/dev setup. AAP/ONTAP/FC omitted. |
| Administration forms, R1/R2, IC-3 | Unit / UI DEV; TC-R1-03 | Extend existing StorageBackendCreatePage.test.tsx and StorageTierCreatePage.test.tsx; `pnpm test` from osac-ui, plus `pnpm run typecheck` and `pnpm lint` | Real forms/proto serialization; mock API hooks; no live proxy/API. |
| Operator input and readiness, R2/R3, IC-4/5/6 | Unit + Envtest / owning DEV; TC-R2-02, TC-R3-01 | Extend `osac-operator/pkg/provisioning/aap_provider_test.go` and `internal/controller/storage_controller_test.go`; proposed `internal/controller/netapp_storage_envtest_test.go`; `make test` from osac-operator | Unit uses fake clients; Envtest uses real K8s/etcd, controlled jobs and API/provider doubles. No deployed AAP or Trident controller. |
| AAP dispatch/state/resource routing, R3/R4, IC-4/5/6 | Component integration / onboarding DEV; TC-R3-02, TC-R4-01 | Extend storage-provider targets in `osac-aap/tests/integration/targets/`; proposed ONTAP fixture target registered in run_tests.sh; existing `STORAGE_TESTS_ENABLED=true make test` from osac-aap | Real Ansible and Kind APIs; proposed ONTAP HTTPS double and TBC status simulator. Existing VMS double does not provide ONTAP coverage. |
| Operator → deployed AAP → ONTAP, R3, IC-4/5/6 | Contract / onboarding DEV; TC-R3-03 | Proposed live contract harness, no committed runner/command yet | Must use real job launch/extra_vars and ONTAP readback; no fake provider. Execution gap belongs to [OSAC-4843](https://redhat.atlassian.net/browse/OSAC-4843) boundary work and this feature's DEV coverage; specific NetApp harness task not yet assigned. |
| Native FC VM acceptance and isolation, R3/R5, IC-5/6 | E2E / QE with OSAC-6037 owner; TC-R5-01/02 | Manual acceptance in prepared lab; proposed automated extension of `tests/e2e/storage/test_tenant_storage_lifecycle.py` and VMaaS suite, runner not established | Real OSAC/AAP/ONTAP/Trident/CDI/KubeVirt/workers/fabric. Requires design §9.1/9.2; [OSAC-6037](https://redhat.atlassian.net/browse/OSAC-6037) shared VM path. No FC support claimed by existing storage-class-only tests. |
| Deployed guarded deletion, R4, IC-5/6 | E2E / QE; TC-R4-02 | Same manual lab acceptance; automate only after live path established | Real data/Trident dependencies and second-tenant preservation; creation/cleanup privileges required. |

## Test Cases

### R1: Register a backend through existing administration interfaces and report access failures

#### TC-R1-01: Validate configuration and bound the management probe

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-1 | critical | automated | Unit | Configuration DEV |

##### Preconditions
Local verified-HTTPS probe double supports success, 401, 403, stalled response and untrusted certificate; a valid password_secret fixture exists.
##### Steps
1. Create ONTAP backends with complete config, missing ONTAP branch/subnet, duplicate FC placement and password/password_secret conflicts; submit ONTAP config with provider=vast.
2. Repeat valid Create against each probe outcome; Update credentials with a partial mask; attempt immutable config change.
##### Expected Results
Valid Create performs one bounded authenticated read against the cluster endpoint and persists READY; it makes no SVM/LIF creation request and does not use a future tenant management address. Missing required config, provider/oneof mismatch and invalid credential choice return InvalidArgument before persistence. Probe 401 returns InvalidArgument; 403 FailedPrecondition; TLS/connect failure Unavailable; timeout DeadlineExceeded with a 10-second request deadline. Credential Update validates merged state and repeats the probe; provider/config mutation is rejected. Responses/logs contain no password or raw auth response.

#### TC-R1-02: CLI configuration survives API persistence

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-1, IC-3 | critical | automated | Component integration | Configuration DEV |

##### Preconditions
Kind/dev fulfillment harness and CLI built from changed source; service can reach the proposed HTTPS probe double.
##### Steps
1. Use existing CLI JSON Create input for a backend with ONTAP config and referenced password Secret.
2. Get/List it, create a linked tier, and retry a rejected immutable masked Update.
##### Expected Results
Get returns the selected ONTAP oneof branch, placement fields and READY state; the JSON config is spec.ontap, without a provider_config wrapper. Tier backend_id references the created object. Update returns InvalidArgument and the stored config is unchanged. An old-provider object with an unset extension remains accepted.

#### TC-R1-03: UI sends NetApp fields and shows registration errors

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-3 | high | automated | Unit | UI DEV |

##### Preconditions
Existing backend/tier form test harness with API-hook doubles and regenerated proto types.
##### Steps
1. Select NetApp ONTAP, fill management/subnet/FC fields, submit; configure a BLOCK tier with native IOPS cap.
2. Return a sanitized registration failure from the hook; switch the backend form to VAST.
##### Expected Results
Payload contains spec.provider=ontap and the generated typed ONTAP oneof branch; tier association contains ontap.maxIops, not reinterpretations of read/write bandwidth. Error is visible and no success navigation occurs. Switching to VAST clears the ONTAP branch; existing VAST tests retain their payload assertions.

### R2: Configure native FC block tiers and preserve the configuration handoff

#### TC-R2-01: Reject incompatible tier semantics

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-2 | critical | automated | Unit | Configuration DEV |

##### Preconditions
Stored ONTAP and VAST backends and existing private-tier handler test harness.
##### Steps
1. Create ONTAP BLOCK tiers with an absent QoS branch, max_iops=0 and 5000.
2. Submit NFS, multiple associations, nonexistent backend, ONTAP QoS on a VAST association, negative cap, nonzero generic bandwidth and immutable masked association updates.
##### Expected Results
Valid tiers retain the native cap; absent/zero QoS is uncapped. API backend lookup rejects the provider/QoS mismatch with InvalidArgument, as well as malformed ONTAP associations; a nonexistent backend returns NotFound. No conversion to an FC enum or generic bandwidth cap occurs. Public tier output omits private provider settings. Existing providers retain their prior validation behavior.

#### TC-R2-02: Serialize one connection and provider-specific tier settings

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-4 | critical | automated | Unit | Configuration DEV |

##### Preconditions
Operator backend/secret API doubles; two tiers reference the same ONTAP backend and have different caps/encryption flags.
##### Steps
1. Resolve definitions and build AAP extra_vars.
2. Inspect the payload using the design's IC-4 fixture and §4.2.1 backend input; repeat with an existing-provider tier, then with a missing credential Secret and API failure.
##### Expected Results
Exactly one storage_backend_connections entry exists, with snake_case ONTAP fields under provider_config and no vendor-named wrapper. Both numeric caps survive under qos_limits.provider_config.max_iops; generic static_limits and boolean encryption_enabled retain their types. The existing-provider payload retains its current fields and omits unused extensions. Missing credentials/API failures produce a non-ready ONTAP outcome without default-class fallback or an empty-password provisioning request. Payload is not emitted to logs.

### R3: Onboard isolated tenants with visible failures and idempotent recovery

#### TC-R3-01: A progress Secret cannot satisfy readiness

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | automated | Envtest | Onboarding/operator DEV |

##### Preconditions
Real Envtest API/etcd with Tenant CRD; controlled AAP job double; ONTAP tenant/config fixture.
##### Steps
1. Persist phase=provisioning, then a phase=ready record with the wrong tenant UID; reconcile.
2. Supply matching complete state; hold class job failed, then complete it and publish labeled classes; restart reconciliation.
3. Delete a tenant with partial owned progress while AAP/connection resolution is unavailable, then restore it.
##### Expected Results
Progress/wrong-UID records never make StorageBackendReady true. Failed class binding does not publish usable tier bindings. Completed matching setup/class stages yield StorageBackendReady and ClusterStorageReady true plus exact name/tier entries. Restart resumes recorded work without a second setup allocation. Deletion finds partial progress independently of readiness; unavailable AAP/connection resolution retains the finalizer until owned cleanup can be verified.

#### TC-R3-02: Run real Ansible dispatch and preserve partial resources

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-4, IC-5, IC-6 | critical | automated | Component integration | Onboarding DEV |

##### Preconditions
Proposed ONTAP HTTPS double and TBC simulator in the existing Kind/Ansible harness; full IC-4 fixture; CSI-install flags disabled.
##### Steps
1. Dispatch setup, inject failure after SVM creation but before management-LIF completion, then retry.
2. Run class stage with TBC failure/timeout, then Bound/Success; inspect Secrets/classes and rerun.
##### Expected Results
Role receives only referenced connections with unchanged provider_config/qos_limits.provider_config contents; generic dispatch does not interpret them. Setup retains full tenant UID/backend ownership and completed steps; retry reuses the SVM and completes the missing LIF. ONTAP failure produces its sanitized reason token. No raw-controller or OSAC CSI installation task runs. No class is published before successful binding; success creates tenant/tier-specific native classes and no credential in tenant namespaces. Replay retains object IDs and the stable tenant_backend_key.

#### TC-R3-03: Verify real AAP-to-ONTAP ownership and allocation

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-4, IC-5, IC-6 | critical | manual | Contract | Onboarding DEV |

##### Preconditions
Live operator/AAP/ONTAP runner agreed under the boundary work; required creation privileges, prepared subnet/ports and native Trident; provider-call failures can be injected without changing unrelated state.
##### Steps
1. Launch normal Tenant storage jobs and read back the SVM, management LIF, FC targets, generated account and native tier policy.
2. Retry after a interrupted/failed job; attempt setup against a same-name SVM with different full ownership.
##### Expected Results
Extra_vars launch reaches the real role; ONTAP objects have recorded SVM/LIF UUIDs and one subnet allocation. The SVM and management LIF use the configured IPspace, including a prepared non-Default fixture if available; aggregate permissions and LIF placements match the registered recipe. Retry preserves identity/address/WWPNs. Ownership mismatch emits OntapOwnershipConflict without mutation. Native capped policy is non-shared and its pool reference matches the tier. Live permission/resource-limit failures leave readiness false with an actionable stage/reason.

### R4: Offboard under dependency guards and preserve other tenants

#### TC-R4-01: Failed cleanup retains the recovery record

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | automated | Component integration | Onboarding DEV |

##### Preconditions
Kind/Ansible harness with ONTAP/TBC doubles and two owned tenant records; one live data dependency or forced cleanup failure.
##### Steps
1. Dispatch teardown with a live workload volume; repeat with missing/unverifiable ownership, wrong Tenant UID and SVM-delete failure.
2. Clear the dependency/failure, resume teardown and replay it.
##### Expected Results
No data-volume force deletion occurs; dependency/failure leaves ownership/credentials and lifecycle finalizer recoverable. NetApp dispatch propagates failure instead of swallowing it and the existing blocking deprovision-job behavior retains the finalizer. Delete receives the Tenant UID and rejects mismatches. Successful order is class removal → TBC/backend disappearance → account revocation and owned array teardown → Secret removal. Replay makes no new resource; second tenant is unchanged. Only verified owned SVM root cleanup is permitted.

#### TC-R4-02: Delete a real tenant only after its disks are gone

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E | QE |

##### Preconditions
Two tenants from TC-R5-01 have working VM disks; approved manual acceptance and real cleanup privileges.
##### Steps
1. Attempt offboarding while tenant A has an active disk; observe guard/status.
2. Remove A's workload/data through the shared lifecycle and confirm data dependencies are gone; offboard A and inspect array/Kubernetes state.
##### Expected Results
Active data blocks teardown with a visible dependency failure. After permitted cleanup, A's TBC/backend, SVM/LIF configuration and credentials are absent and old management access fails. A's finalizer clears after verified cleanup; B's VM still reads its saved data. No full nonempty-SVM force deletion occurs.

### R5: Consume only the tenant's ready tiers through the shared VM path

#### TC-R5-01: Two tenants use distinct SVMs and persist VM disk data over FC

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E | QE with OSAC-6037 owner |

##### Preconditions
Prepared FC lab; no OSAC CSI deployment; agreed native Trident/shared VM route; eligible importer/VM workers have working HBA/fabric paths and new target zoning.
##### Steps
1. Register one backend/two tiers and onboard tenants A/B normally; compare SVM/management IP/target ownership and bindings.
2. Create a VM with a NetApp-backed DataVolume through the normal OSAC interface, write a known value to its disk, restart the VM and read it.
##### Expected Results
Distinct owned SVMs/management LIFs/IPs exist; FC LIFs have SVM-scoped WWPNs, with no tenant IP data LIF. Each class selects only its tenant/backend/tier pool. DataVolume/PVC binds, the shared Volume identity is linked as defined by OSAC-6037, and the VM reads the same saved value after restart. Readiness alone is not accepted as FC I/O evidence; no manual per-VM disk bypass is used.

#### TC-R5-02: Cross-tenant class/access attempts are denied

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E | QE |

##### Preconditions
TC-R5-01 environment and tenant A identity; native Kubernetes permissions/admission boundary identified in design §9.3.
##### Steps
1. Attempt A's VM/storage request selecting B's binding; if A can create native PVCs, attempt B's StorageClass directly.
2. Attempt to read B's native/config credential Secrets; inspect igroup/LUN mappings for unauthorized initiator WWPNs.
##### Expected Results
OSAC/admission denies cross-tenant selection before a B-backed claim is provisioned; any accepted request is an acceptance failure. A cannot read either credential Secret. LUN mappings include only authorized trusted worker initiators; an unauthorized initiator has no mapped LUN. Shared trusted workers are not claimed to be per-tenant physical initiators.

## Gaps

### Requirement Coverage Gaps
All five PRD source anchors have planned cases. Executable live Contract and FC E2E coverage is unresolved: access has been handed off, but infrastructure/privileges, native strategy and OSAC-6037 integration are not validated. No implementation tests ran during design drafting.

### Interface Change Coverage Gaps
All six ICs have planned cases. Planning does not establish an execution path for TC-R3-03 or the deployed FC cases; the matrix records those gaps and responsible workstreams. Native cross-tenant PVC/class enforcement (§9.3) must be verified/assigned before TC-R5-02 can pass.

## Summary

| Metric | Count |
|---|---|
| Total | 12 |
| Critical / High / Medium / Low | 11 / 1 / 0 / 0 |
| Automated / Manual | 8 / 4 |
| Requirements with cases | 5 / 5 local PRD anchors |
| Interface changes with cases | 6 / 6 |
| Executed | 0 |
