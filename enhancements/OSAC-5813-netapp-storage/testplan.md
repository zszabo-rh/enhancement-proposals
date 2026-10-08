# Testplan — OSAC-5813

## Overview

- Feature: [OSAC-5813](https://redhat.atlassian.net/browse/OSAC-5813), NetApp tenant onboarding/offboarding.
- Design: [design.md](design.md), proposed contracts IC-1–IC-6.
- Total: 12 cases; 5/5 PRD source anchors and 6/6 interface changes mapped. R1–R5 are local references to existing unnumbered PRD text, defined in [design §5](design.md#5-interface-changes), not new requirements.
- Status: planned, not executed. Automated classifications describe intended tests. Live ONTAP/AAP/FC execution, prepared resources and handoff agreement remain prerequisites.

## Planning evidence / execution paths

This matrix follows [Integration testing](https://github.com/osac-project/osac/blob/c8d0d8890dc0e381f70a43ab8572c6b03b429565/docs/INTEGRATION-TESTING.md). DEV owns Unit, Envtest, component integration and Contract; QE owns deployed E2E. Existing fixture tests do not prove a live NetApp boundary.

| Component / behavior / refs | Tier / owner / cases | Suite and execution | Real dependencies / doubles / gaps |
|---|---|---|---|
| Fulfillment ONTAP validation/probe, R1/R2, IC-1/2 | Unit / configuration DEV; TC-R1-01, TC-R2-01 | Extend `fulfillment-service/internal/servers/private_storage_{backends,tiers}_server_test.go`; `ginkgo run internal/servers` from fulfillment-service | Real handlers/validation; local HTTPS ONTAP double for probe; not a real array. |
| CLI → API → persistence, R1, IC-1/3 | Component integration / configuration DEV; TC-R1-02 | Proposed `fulfillment-service/it/it_netapp_storage_configuration_test.go`; existing command `make -C ../osac-installer test PLATFORM=kind PROFILE=dev NS=osac SUITE=fulfillment` from fulfillment-service | Real CLI/service/PostgreSQL; proposed reachable HTTPS probe double; requires installer Kind/dev setup. AAP/ONTAP/FC omitted. |
| Administration forms, R1/R2, IC-3 | Unit / UI DEV; TC-R1-03 | Extend existing StorageBackendCreatePage.test.tsx and StorageTierCreatePage.test.tsx; `pnpm test` from osac-ui, plus `pnpm run typecheck` and `pnpm lint` | Real forms/proto serialization; mock API hooks; no live proxy/API. |
| Fulfillment storage-condition projection, R3, IC-6 | Unit / fulfillment DEV; TC-R3-01 | Extend `fulfillment-service/internal/controllers/tenant/tenant_reconciler_function_test.go`; `ginkgo run -r internal` from fulfillment-service | Real condition mapping with fixture CR status; no deployed operator/API feedback boundary. TC-R3-03 covers the proposed live boundary. |
| Operator input and readiness, R2/R3, IC-4/5/6 | Unit + Envtest / owning DEV; TC-R2-02, TC-R3-01 | Extend `osac-operator/pkg/provisioning/aap_provider_test.go` and `internal/controller/storage_controller_test.go`; proposed `internal/controller/netapp_storage_envtest_test.go`; `make test` from osac-operator | Unit uses fake clients; Envtest uses real K8s/etcd, controlled jobs and API/provider doubles. No deployed AAP or Trident controller. |
| AAP dispatch/state/resource routing, R3/R4, IC-4/5/6 | Component integration / onboarding DEV; TC-R3-02, TC-R4-01 | Extend storage-provider targets in `osac-aap/tests/integration/targets/`; proposed ONTAP fixture target registered in run_tests.sh; existing `STORAGE_TESTS_ENABLED=true make test` from osac-aap | Real Ansible and Kind APIs; proposed ONTAP HTTPS double and TBC status simulator. Existing VMS double does not provide ONTAP coverage. |
| Operator → fulfillment feedback / deployed AAP → ONTAP, R3, IC-4/5/6 | Contract / onboarding DEV; TC-R3-03 | Proposed live contract harness, no committed runner/command yet | Must use real job launch/extra_vars and ONTAP readback; no fake provider. Execution gap belongs to [OSAC-4843](https://redhat.atlassian.net/browse/OSAC-4843) boundary work and this feature's DEV coverage; specific NetApp harness task not yet assigned. |
| Native FC VM acceptance and isolation, R3/R5, IC-5/6 | E2E / QE with OSAC-6037 owner; TC-R5-01/02 | Manual acceptance in prepared lab; proposed automated extension of `tests/e2e/storage/test_tenant_storage_lifecycle.py` and VMaaS suite, runner not established | Real OSAC/AAP/ONTAP/Trident/CDI/KubeVirt/workers/fabric. Requires design §9.1/9.2/9.3/9.5; [OSAC-6037](https://redhat.atlassian.net/browse/OSAC-6037) shared VM path, agreed management/account model and two tenant VMs on the same worker/WWPNs. No FC support claimed by existing storage-class-only tests. |
| Deployed guarded deletion, R4, IC-5/6 | E2E / QE; TC-R4-02 | Same manual lab acceptance; automate only after live path established | Real data/Trident dependencies and second-tenant preservation; prepared-resource retention and manual-release handoff required. |

## Test Cases

### R1: Register a backend through existing administration interfaces and report access failures

#### TC-R1-01: Validate configuration and bound the management probe

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-1 | critical | automated | Unit | Configuration DEV |

##### Preconditions
Local verified-HTTPS probe double supports success, 401, 403, stalled response and untrusted certificate; a valid password_secret fixture exists.
##### Steps
1. Create ONTAP backends using only common endpoint/credentials; submit malformed endpoint and password/password_secret conflicts. No physical configuration branch exists.
2. Repeat valid Create against each probe outcome, including discovery-read denial; Update credentials with a partial mask; attempt immutable endpoint change.
##### Expected Results
Valid Create completes bounded authenticated management/discovery reads against the cluster endpoint and persists READY, even before tenant SVMs exist. No array mutation occurs. Invalid endpoint/credential choice returns InvalidArgument before persistence. Probe 401 returns InvalidArgument; 403 FailedPrecondition; TLS/connect failure Unavailable; timeout DeadlineExceeded under the proposed 10-second total deadline. Credential Update validates merged state and repeats the probe; provider/ONTAP endpoint mutation is rejected. Responses/logs contain no password or raw auth response.

#### TC-R1-02: CLI configuration survives API persistence

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-1, IC-3 | critical | automated | Component integration | Configuration DEV |

##### Preconditions
Kind/dev fulfillment harness and CLI built from changed source; service can reach the proposed HTTPS probe double.
##### Steps
1. Use existing CLI JSON Create input for an ONTAP backend with common endpoint and referenced password Secret.
2. Get/List it, create a linked tier, and retry a rejected immutable masked Update.
##### Expected Results
Get returns provider=ontap, the common endpoint/credential reference and READY state; no placement/provider_config branch is needed. Tier backend_id references the created object. ONTAP endpoint Update returns InvalidArgument and the stored connection is unchanged. An existing-provider backend remains accepted.

#### TC-R1-03: UI sends NetApp fields and shows registration errors

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-3 | high | automated | Unit | UI DEV |

##### Preconditions
Existing backend/tier form test harness with API-hook doubles and regenerated proto types.
##### Steps
1. Select NetApp ONTAP, fill existing endpoint/credentials and submit; configure a BLOCK tier with native IOPS cap.
2. Return a sanitized registration failure from the hook; switch the backend form to VAST.
##### Expected Results
Backend payload contains provider=ontap and existing connection fields, with no placement fields. Tier association contains its typed ontap.maxIops branch, not a bandwidth reinterpretation. Registration error is visible and no success navigation occurs. Switching provider clears incompatible tier QoS; existing-provider payload assertions remain.

### R2: Configure native FC block tiers and preserve the configuration handoff

#### TC-R2-01: Reject incompatible tier semantics

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-2 | critical | automated | Unit | Configuration DEV |

##### Preconditions
Stored ONTAP and VAST backends and existing private-tier handler test harness.
##### Steps
1. Create ONTAP BLOCK tiers with an absent QoS branch, max_iops=0 and 5000.
2. Submit NFS, multiple associations, nonexistent backend, ONTAP QoS on a VAST association, negative cap, nonzero generic bandwidth and immutable masked association/QoS/encryption updates.
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
Exactly one storage_backend_connections entry exists with common endpoint/discovery credentials and no physical recipe or SVM password. Both numeric caps survive under qos_limits.provider_config.max_iops; generic static_limits and boolean encryption_enabled retain their types. The existing-provider payload retains its current fields and omits unused extensions. Missing credentials/API failures produce a non-ready ONTAP outcome without default-class fallback or an empty-password provisioning request. Payload is not emitted to logs.

### R3: Onboard isolated tenants with visible failures and idempotent recovery

#### TC-R3-01: A progress Secret cannot satisfy readiness

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | automated | Unit + Envtest | Onboarding/operator and fulfillment DEV |

##### Preconditions
Real Envtest API/etcd with Tenant CRD; controlled AAP job double; prepared-SVM ownership fixture. Unit target-client fixtures supply separate hub/remote Namespace UIDs and class lists. Fulfillment unit harness separately checks condition projection.
##### Steps
1. Persist phase=validating, then a phase=ready record with the wrong tenant UID/source generation; reconcile.
2. Supply matching complete state; hold class job failed, then complete it and publish labeled classes; restart reconciliation.
3. Delete a tenant with partial owned progress while AAP/connection resolution is unavailable, then restore it. Unit-check storage-condition projection with true/false reasons and missing conditions.
4. Unit-check complete records with mismatched target mode/cluster UID; provide classes only on the hub while the configured hosting target is remote.
##### Expected Results
Progress/wrong-UID records never make StorageBackendReady true. Failed class binding does not publish usable tier bindings. Completed matching setup/class stages yield StorageBackendReady and ClusterStorageReady true plus exact name/tier entries. Restart resumes recorded work without a second claim or native-resource allocation. Fulfillment preserves both private storage condition types/reasons separately from IDP status; absent conditions never imply ready. Deletion finds partial progress independently of readiness; unavailable AAP/connection resolution retains the finalizer until owned cleanup can be verified.
Recorded target mode/cluster UID mismatches cannot establish StorageBackendReady. Hub-local classes cannot establish ClusterStorageReady for dedicated hosting; discovery uses the configured target client.

#### TC-R3-02: Run real Ansible dispatch and preserve the prepared assignment

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-4, IC-5, IC-6 | critical | automated | Component integration | Onboarding DEV |

##### Preconditions
Proposed ONTAP HTTPS double and TBC simulator in the existing Kind/Ansible harness; full IC-4 fixture; CSI-install flags disabled. Exercise hub hosting and a second Kind cluster as the configured dedicated VM target; storage jobs retain hub access for recovery records.
##### Steps
1. Prepare matching SVM/source Secret/native policy fixtures. Dispatch setup with native driver prerequisites missing, then present; inject interruption after claim but before ready-state persistence, then retry. Try missing preparation, mismatched UUID and competing UID.
2. Run class stage with TBC failure/timeout, then Bound/Success; inspect Secrets/classes and rerun. Submit a credential/endpoint scope mismatch under the agreed management model, a missing source Secret and denied native-namespace access.
3. Repeat setup/class actions with the remote target configured. Make its kubeconfig unreadable, then restore it; inspect source claims, hub-state placement and native resources.
##### Expected Results
Role receives only referenced common connections and unchanged qos_limits.provider_config contents. Repeated backend/tenant inputs derive the same lookup names. Missing native driver prerequisites fail before claiming storage; no driver-install task runs. Setup validates prepared resources, atomically claims full tenant UID/backend/SVM UUID/source UID and persists progress; retry resumes the same claim. Missing/mismatched preparation fails without creating SVMs/LIFs/accounts/policies. No class is published before successful binding. Native configuration combines discovered SVM/endpoint, role driver/protocol/version defaults, resolved tier intent and the administrator-prepared Secret reference. Its credentials.name points to the source in the native namespace, with no embedded password or substitution of discovery credentials. Missing/denied Secret access and endpoint/account mismatches remain not ready with sanitized errors. No credential is copied to a tenant namespace. Replay preserves IDs and assignment_key.
In dedicated mode source claim, TBC and StorageClass operations use the remote API; the progress record is written on the hub with the selected target mode/cluster UID. No native resources are created on the hub. Unreadable configured remote credentials fail rather than switching target. Operator readiness/target discovery is checked in TC-R3-01 and deployed acceptance.

#### TC-R3-03: Verify real AAP-to-ONTAP adoption and status feedback

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-4, IC-5, IC-6 | critical | manual | Contract | Onboarding DEV |

##### Preconditions
Live operator/fulfillment/AAP/ONTAP runner agreed under the boundary work; two prepared SVMs, source credentials, matching native policies and native Trident. Read-only discovery permissions and the selected native endpoint/account privileges are verified. Per-SVM management is the proposed baseline; a shared endpoint is exercised only if design §9.5 selects and qualifies it.
##### Steps
1. Launch normal Tenant storage jobs; compare prepared SVM/LIF/account/policy identities before/after and read storage conditions through the private API.
2. Retry after interruption; attempt adoption with mismatched UUID/ownership, unclaimed workload data and missing source credentials/policy.
##### Expected Results
Real launch reaches the role and its claim/native backend; existing SVM/LIF/account/policy UUIDs, addresses and WWPNs are unchanged. Native TBC uses the approved endpoint/account scope and explicit SVM identity; provisioning never falls back to the read-only discovery identity. Native capped policy is SVM-scoped/non-shared with the requested cap and matching pool reference. Discovery/account failures remain not ready with sanitized reasons. Conflict/previous data rejects adoption before native configuration. Retry reuses the claim. Private API conditions reflect both storage stages and failures; SYNCED IDP state alone never establishes storage readiness.

### R4: Offboard under dependency guards and preserve other tenants

#### TC-R4-01: Failed cleanup retains the recovery record

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | automated | Component integration | Onboarding DEV |

##### Preconditions
Kind/Ansible harness with ONTAP/TBC doubles, prepared-source Secrets and two ownership records; one live data dependency or forced native cleanup failure.
##### Steps
1. Dispatch teardown with live workload data; repeat with unverifiable ownership, wrong Tenant UID and TBC/backend deletion failure.
2. Clear the dependency/failure, resume teardown and replay it. Attempt same-name/new-UID adoption, source deletion alone and then a newly authorized source generation after cleanup.
3. Repeat guarded cleanup with the dedicated target; change the target connection while an assignment is active and attempt teardown.
##### Expected Results
No data-volume force deletion occurs; dependency/failure leaves ownership/credentials and lifecycle finalizer recoverable. NetApp dispatch propagates failure instead of swallowing it and the existing blocking deprovision-job behavior retains the finalizer. Delete receives the Tenant UID and rejects mismatches. Successful order is owned class removal → TBC/backend disappearance → assignment/source marked retained → finalizer completion. SVM/LIF/account/policy/source and retained record remain. Another UID or source deletion alone cannot release the claim. Only a retained record, new source generation and verified dependency/data cleanup allow authorized reassignment. Replay is idempotent; the second tenant is unchanged. No array teardown occurs.
Dedicated cleanup removes owned native resources only from the recorded hosting target and updates the recovery record on the hub. A changed/unavailable target blocks cleanup and retains recovery state; it never deletes matching names on a different cluster.

#### TC-R4-02: Delete a real tenant only after its disks are gone

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E | QE |

##### Preconditions
Two tenants from TC-R5-01 have working VM disks and prepared resources; approved manual acceptance and native cleanup permissions.
##### Steps
1. Attempt offboarding while tenant A has an active disk; observe guard/status.
2. Remove A's workload/data through the shared lifecycle and confirm data dependencies are gone; offboard A and inspect array/Kubernetes state.
##### Expected Results
Active data blocks teardown with a visible dependency failure. After permitted cleanup, A's owned classes/TBC/backend are absent, departing OSAC access is invalidated and its claim is retained. Prepared SVM/LIF/account/policy/source identities remain. Recreating A's name does not reuse the old claim automatically. Finalizer clears after verified OSAC cleanup; B's VM still reads saved data. Infrastructure admins own later data/access cleanup and explicit release.

### R5: Consume only the tenant's ready tiers through the shared VM path

#### TC-R5-01: Two tenants share FC worker initiators and persist isolated VM disk data

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E | QE with OSAC-6037 owner |

##### Preconditions
Prepared FC lab; no OSAC CSI deployment; agreed native Trident/shared VM route and management/account model; eligible importer/VM workers have working HBA/fabric paths and prepared target zoning; two dedicated SVMs/accounts/policies/source Secrets are ready. Tenant A/B VMs can run on the same worker, whose HBA WWPNs are authorized for both SVMs under the supported igroup/LUN model. Run this case for hub hosting and dedicated remote hosting, each with one configured target; source Secrets/native Trident are installed on that target.
##### Steps
1. Register one backend/two tiers, prepare dedicated SVMs by convention and onboard tenants A/B normally; compare source/SVM/management IP/target ownership and bindings.
2. Create A/B VMs with NetApp-backed DataVolumes through the normal OSAC interface, place them on the same FC-connected worker and inspect the shared host WWPN/igroup/LUN mappings and guest disk assignments.
3. Correlate each private OSAC Volume record with its DataVolume/PVC and native allocation. Write distinct known values to A/B disks, restart both VMs and read them.
4. For dedicated hosting, inspect both clusters: Tenant/status/progress remain on the hub; native bindings and VM disk objects are on the configured remote cluster.
##### Expected Results
Distinct prepared SVMs are adopted without infrastructure recreation. Management endpoints/accounts match the approved model; the proposed baseline has distinct management LIFs/IPs, while a qualified shared cluster endpoint retains explicit SVM selection. FC LIFs have SVM-scoped target WWPNs, with no tenant IP data LIF. Each class selects only its tenant/backend/tier pool. Both SVMs support the same trusted worker initiator WWPNs with correct native LUN mappings; A/B guests receive only their authorized disks. A private Volume record precedes each DataVolume; PVC/Trident performs physical provisioning without a second allocation triggered by OSAC bookkeeping. Shared Volume identity/status/cleanup follows OSAC-6037. Each VM reads its own saved value after restart. Readiness alone is not FC I/O evidence; no manual per-VM disk bypass is used.
In dedicated mode FC I/O comes from hosting-cluster workers; the hub needs management connectivity but no worker HBA merely to run OSAC. TBC, credential sources, StorageClasses, DataVolumes/PVCs/PVs and VMs are on the remote target. Ready bindings and guarded offboarding work through the hub API; no hub-local class substitutes for a missing remote class. Both deployment modes pass the disk persistence/isolation checks.

#### TC-R5-02: Cross-tenant class/access attempts are denied

| Interface Change | Priority | Automation | Tier | Owner |
|---|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E | QE |

##### Preconditions
TC-R5-01 shared-worker environment and tenant A identity; native Kubernetes permissions/admission and trusted-host guest-device boundary identified in design §9.3.
##### Steps
1. Attempt A's VM/storage request selecting B's binding; if A can create native PVCs, attempt B's StorageClass directly.
2. Attempt to read B's native/config credential Secrets; inspect igroup/LUN mappings for unauthorized initiator WWPNs.
3. From A's guest, inspect accessible disks and attempt access to B's test disk/data; repeat from B toward A and compare with host-side device assignment.
##### Expected Results
OSAC/admission denies cross-tenant selection before a B-backed claim is provisioned; any accepted request is an acceptance failure. A cannot read either credential Secret. LUN mappings include only authorized trusted worker initiators; an unauthorized initiator has no mapped LUN. The shared trusted worker may access both tenants' mapped LUNs, but each guest can access only its authorized disk/data. Cross-guest access is an acceptance failure. Shared trusted workers are not claimed to be per-tenant physical initiators; separate SVMs/classes alone are not accepted as proof.

## Gaps

### Requirement Coverage Gaps
All five PRD source anchors have planned cases. Executable live Contract and FC E2E coverage is unresolved: access has been handed off, but prepared resources/privileges, selected management/account scope, shared-WWPN/guest isolation, credential/release conventions and OSAC-6037 bookkeeping integration are not validated. The MVP uses named preassignment; hub and dedicated remote hosting are required deployment modes. A dedicated FC target and cross-cluster credential/state/cleanup routing must be verified before acceptance. No implementation tests ran during design drafting.

### Interface Change Coverage Gaps
All six ICs have planned cases. Planning does not establish an execution path for TC-R3-03 or the deployed FC cases; the matrix records those gaps and responsible workstreams. Native cross-tenant PVC/class enforcement, shared worker FC mappings and guest device isolation (§9.3) must be verified/assigned before TC-R5-01/02 can pass. Management/credential selection (§9.5) remains an execution prerequisite, not a requirement to implement both modes.

## Summary

| Metric | Count |
|---|---|
| Total | 12 |
| Critical / High / Medium / Low | 11 / 1 / 0 / 0 |
| Automated / Manual | 8 / 4 |
| Requirements with cases | 5 / 5 local PRD anchors |
| Interface changes with cases | 6 / 6 |
| Executed | 0 |
