---
title: netapp-shared-trident-preview
authors:
  - zszabo@redhat.com
creation-date: 2026-10-06
last-updated: 2026-10-16
tracking-link: https://redhat.atlassian.net/browse/OSAC-5813
prd: prd.md
---

# NetApp preview: shared Trident backend and tenant tiers

## Summary

OSAC registers an administrator-managed shared TridentBackendConfig (TBC) and
creates tenant/tier StorageClasses selecting its prepared virtual pools. The
administrator owns native configuration; Trident handles array connectivity.
See [PRD](prd.md) for detailed requirements.

## Motivation

Existing storage reconciliation already separates backend setup from tenant
class creation. Shared native storage gives the first stage a useful role:
observe the prepared backend before publishing usable classes. No hub-side
ONTAP connection, per-tenant SVM claim or credential copy is needed.

### Goals

- Extend existing registration APIs with typed, named backend profiles and tier pool selection.
- Preserve existing providers, lifecycle stages and current-configuration readiness gates.
- Keep customer-owned native resources outside OSAC mutation and cleanup.

### Non-Goals

- Dedicated storage mode, per-tenant SVM/TBC provisioning or automatic pool allocation.
- Array discovery/probes, credential rotation or native TLS-policy management.
- Driver installation, new hosting-cluster routing, UI implementation or Enclave.
- Shared VM/Volume lifecycle implementation, independent VaaS, CaaS/BMaaS or non-FC transports.

## Proposal

`StorageBackend.spec.ontap` supplies named profiles referencing existing TBCs.
`StorageTier.spec.backends[].ontap` selects a profile and virtual-pool labels.
The operator passes references/selectors to AAP, without resolving credentials.
The ONTAP role observes the TBC and creates only OSAC-owned tenant classes.

### Workflow Description

Registration and Tenant onboarding are independent. The only required ordering
is that the needed native backend is usable when its AAP setup stage executes.

| Phase / trigger | Actor and action | Output |
|---|---|---|
| Infrastructure preparation | Cloud Infrastructure Admin prepares SVM, FC workers/fabric, Trident, native credentials/trust, shared TBC and virtual-pool QoS/encryption | Usable native backend on the VM hosting cluster |
| Backend Create | Cloud Provider Admin supplies common required fields and a named shared TBC profile via CLI/API | Catalog READY; no array request |
| Tier Create | Cloud Provider Admin chooses backend ID, profile and pool labels | Stored FC block offering |
| Tenant becomes Ready | Existing storage operator resolves catalog and starts backend AAP job | Setup pending; no preparation Secret lookup |
| Backend setup succeeds | AAP observes selected TBCs through target Kubernetes API | `StorageBackendReady=True` for current configuration |
| Class setup succeeds | AAP creates tenant/tier classes; operator observes expected classes | `ClusterStorageReady=True`, `{name, tier}` bindings |
| Tenant offboarding | Existing guarded lifecycle runs class cleanup then backend bookkeeping cleanup | Only departing tenant's OSAC configuration removed |

The hub runs the operator and AAP. Trident/TBC, classes and VM disks belong to
one deployment-selected hosting target: the hub or a dedicated VM cluster.
Existing remote kubeconfig/target-client configuration is reused. TBC namespace
and name do not identify a different cluster. AAP and operator class observation
must use the same selected target; unavailable remote configuration fails closed.
ONTAP management addresses remain provider infrastructure, with no new OSAC
Subnet, ExternalIP or guest-network attachment behavior.

### API Extensions

Existing Create/Get/List/Update APIs, DB JSON storage and Tenant CR shape remain.
No new SQL table, StorageBackend CRD, backend reconciler or imperative retry API.

#### Interface Changes

R1–R5 are local references to existing unnumbered PRD text, also used by the
[test plan](testplan.md); they are not new product requirements.

| Anchor | PRD source |
|---|---|
| R1 | Backend registration — In Scope; Cloud Provider Admin story |
| R2 | FC tiers and prepared policies — In Scope; Cloud Provider Admin story |
| R3 | Onboarding, readiness and retry — Required verification; admin/tenant stories |
| R4 | Guarded cleanup and shared-resource retention — In Scope; Required verification |
| R5 | Shared VM consumption and isolation — Required verification; Dependencies |

| Interface | Change / requirement anchors |
|---|---|
| IC-1 Backend | Typed ONTAP profiles and configuration-only registration; R1/R3 |
| IC-2 Tier | Named profile plus literal pool-selector map; R2/R3 |
| IC-3 Administration | Existing generic CLI/API accepts the fields; UI team owns form support; R1/R2 |
| IC-4 Operator → AAP | Generic backend/tier `provider_config`; no shared-path credentials/preparation; R2/R3 |
| IC-5 AAP → consumption | Existing provider output/class binding shapes; customer TBC retained; R3/R4/R5 |
| IC-6 Lifecycle | Preserve two readiness stages, version checks and ordered guarded cleanup; R3/R4/R5 |

### Implementation Details/Notes/Constraints

#### Backend profile and tier schema

The existing `StorageBackendSpec` adds field 5 using the established typed-oneof
pattern. ProtoJSON exposes `ontap` directly, without a `providerConfig` wrapper.

```protobuf
oneof provider_config { OntapBackendConfig ontap = 5; }
message OntapBackendConfig { repeated OntapBackendProfile profiles = 1; }
message OntapBackendProfile {
  string name = 1;
  oneof mode { SharedTridentBackend shared = 2; }
}
message SharedTridentBackend {
  string tbc_namespace = 1;
  string tbc_name = 2;
}
```

Profiles are unique DNS-label names, 1–32 per backend; shared mode is required
and validates namespace/TBC name. A later dedicated branch can extend the mode
oneof. No dedicated mode is implemented or accepted for this preview.

Common endpoint/credentials stay required, including exactly one password or
VALUE password reference, pending a separate decision to relax them. They remain
stored for compatibility; shared onboarding does not read/send their values.
Secret references are checked for visibility/type/lifecycle using metadata.
Registration validates locally and stores READY without ONTAP access. Backend
provider, ONTAP endpoint and profiles are immutable; description/credentials
remain editable through existing update-mask/locking rules. Accepted Update
clears historical probe ERROR/message. [User]

The existing `OntapAssociationConfig` retains `max_iops = 1` and adds
`profile = 2` and `pool_selector = 3`. Shared associations require one ONTAP
backend, BLOCK, a known profile and a non-empty selector. Numeric IOPS and
generic bandwidth must be zero/absent; provider QoS comes from the prepared pool.
Selectors are bounded literal key/value maps, rejecting expression delimiters.
Tier binding immutability and public/private visibility remain unchanged.

For a real deployment, `trident/customer-fc` might expose virtual pools labelled
`performance=extreme` and `performance=value`. The administrator's TBC supplies
the corresponding native adaptive QoS and encryption defaults. OSAC registers:

```yaml
# StorageBackend.spec, alongside required endpoint/credentials:
provider: ontap
ontap:
  profiles:
    - name: shared-fc
      shared: {tbcNamespace: trident, tbcName: customer-fc}
```

```yaml
# StorageTier.spec.backends[0]:
backendId: registered-backend-id
encryptionEnabled: true
ontap:
  profile: shared-fc
  poolSelector: {performance: extreme}
```

`encryption_enabled` remains advertised tier metadata. It is not an instruction
to edit the shared TBC or override native settings. The administrator qualifies
that the chosen pool provides the advertised encryption; OSAC does not select
NAE/NVE or read the array to establish that result. Native volume verification
remains an acceptance check. [User]

#### Operator/AAP handoff

The operator deduplicates connections by backend ID. Shared profiles skip OSAC
password resolution and hub preparation lookup, while retaining readiness gates.
The exact payload is documented in the
[onboarding interface](https://github.com/zszabo-rh/osac/blob/OSAC-5813-netapp-backend-tiers/docs/guides/developer/netapp-onboarding-contract.md).
Its key parts are:

```yaml
storage_backend_connections:
  backend-id:
    provider_config:
      profiles:
        - name: shared-fc
          shared: {tbc_namespace: trident, tbc_name: customer-fc}
storage_tier_definitions:
  - name: fast
    provider: ontap
    protocol: block
    backend_id: backend-id
    encryption_enabled: true
    qos_limits:
      static_limits: {max_reads_bw_mbps: 0, max_writes_bw_mbps: 0}
    provider_config:
      profile: shared-fc
      pool_selector: {performance: extreme}
```

The shared connection omits endpoint/user/password/tenant_preparation entirely.
The generic serializer still supports common fields alongside provider config
for other providers. Existing dispatch passes whole dictionaries to the role.

#### AAP receiver: implementation boundary

The onboarding receiver must implement the following; this branch verifies the sender,
not these native operations:

1. `setup`: select each referenced profile, read the existing TBC on the hosting
   cluster, require Bound/Success and the FC configuration, and publish the
   existing `storage_provider_tenant_config` output. No ONTAP REST call.
2. `ensure_storage_class`: resolve `status.backendInfo.backendName`, then create
   tenant/tier classes using `csi.trident.netapp.io`, the exact native backend
   restriction and selected pool labels. Publish existing class-name outputs.
3. `teardown_cluster_storage`: respect data/ownership guards; remove only the
   departing tenant/provider's classes and OSAC-owned configuration.
4. `teardown_backend`: remove only OSAC bookkeeping. Never mutate/delete the
   customer TBC, native credential Secret, SVM or pool/policy configuration.

TBC metadata.name and native backendName may differ. Native class selection must
restrict both backend and pool; selectors alone could match another backend.
Render a safely escaped exact backend filter and deterministic literal selector
entries. Keep existing lifecycle defaults and `{name, tier}` output conventions.
The native FC TBC uses `ontap-san` / `sanType: fcp`; customer preparation owns that
setting. [TBC status](https://docs.netapp.com/us-en/trident-2510/trident-use/backend-kubectl.html),
[virtual pools](https://docs.netapp.com/us-en/trident-2510/trident-use/ontap-san-examples.html).

### Security Considerations

Existing provider-admin authorization protects backend/tier details. Tenant
responses contain class/tier bindings, not native credentials. Only Trident
connects to ONTAP; authentication and server trust are configured in the native
backend. Removing hub probes does not disable Trident's certificate validation.
The required common backend password is unused on the shared path and is not
copied into AAP variables. Existing credential-bearing legacy paths retain redaction.

A shared SVM/TBC does not provide per-tenant SVM isolation. OSAC authorization,
class access controls and trusted-worker guest-device assignment must enforce
the tenant boundary; a pool label or FC worker WWPN alone does not. Native
cross-tenant selection/device-access checks remain release acceptance work.

### Failure Handling and Recovery

| Failure | Observable behavior / recovery |
|---|---|
| Invalid API fields/reference or unknown profile | API rejects without saving; admin corrects input |
| Catalog unavailable or profile cannot resolve | Both storage conditions false, `ConfigurationUnavailable`; no dispatch; requeue |
| Missing/unbound/unusable TBC | AAP setup fails; storage stays not ready; admin fixes native preparation and existing retries resume |
| Class job failure or missing expected class | No cluster-storage readiness; retry/repair is idempotent |
| Deletion job failure or unavailable cleanup routing | Retain finalizer; restore routing/resolve dependency and retry |

Backend catalog READY is distinct from Tenant `StorageBackendReady`. The latter
requires a successful setup job for the current configuration; class readiness
also requires successful class setup and all observed tier classes. Versions
include profile/TBC routing and selectors, so stale completed jobs cannot satisfy
changed desired configuration. Merely finding an input object does not mean ready.

The operator persists existing job history/versioned progress. Restart resumes
that lifecycle; it does not create a per-tenant SVM claim. AAP must inspect/update
only fully owned tenant output and tolerate repeated class operations. No new
continuous TBC/array-health watch is introduced. Fulfillment projects both storage
conditions independently, including reasons and generation checks.

### RBAC / Tenancy

No new API authorization or operator TBC-read permission is needed. AAP requires
read access to the referenced TBC and the existing class/configuration permissions
on the hosting target. OSAC-owned resources preserve tenant and owner-reference
annotations, discovery labels and full UID ownership. Shared customer resources
must have no per-tenant garbage-collection relationship. Cleanup and class access
must not rely on an unscoped tenant-name selector alone.

### Observability and Monitoring

Reuse storage conditions, existing AAP job history and Kubernetes/Trident events.
No new metrics or health-monitor service. Report stage/backend/profile and
sanitized failure reasons; do not dump native Secret contents.

### Risks and Mitigations

- Sender/receiver skew: deploy matched revisions before admitting shared profiles.
- Wrong pool/backend selection: constrain both, then test two tenants and tiers.
- Shared-resource deletion: observe native inputs only and verify retention on retry/offboarding.
- Advertised policy mismatch: administrator qualification and native volume readback.

### Drawbacks

Manual native preparation remains substantial, and catalog READY cannot diagnose
array connectivity. Shared native storage puts tenant disk separation on the
hosting platform. These constraints match the preview scope and avoid giving
hub services unnecessary array credentials or provisioning responsibilities.

## Alternatives (Not Implemented)

| Alternative | Benefit / reason not selected |
|---|---|
| Per-tenant SVM/TBC creation | Stronger native separation; requires a larger credential/provisioning lifecycle outside shared preview scope |
| Hub ONTAP connectivity probe | Early diagnostic; tests the wrong network location and adds unnecessary credentials/trust |
| Environment-only profile/pool mapping | Faster temporary configuration; lacks explicit catalog validation and stable persisted associations |
| Opaque Struct/Any provider settings | Flexible; weaker generated validation and discoverability than typed oneofs |
| Oneof-free provider fields | Easy initially; permits conflicting provider configurations as vendors are added |
| Single combined setup/class job | Fewer stages; loses existing progress/readiness separation and changes shared lifecycle patterns |

## Open Questions

### Can common endpoint/credentials become optional for shared profiles?

**Owner:** Architects / API owners. **Impact:** they remain mandatory until agreed;
relaxation must preserve other providers' requirements. No shared-path runtime use.

### Is the native shared VM path qualified for the preview environment?

**Owner:** Onboarding / shared VM / infrastructure / QE. **Impact:** complete
receiver tests, FC persistence, advertised encryption, tenant isolation and shared
resource retention. See PRD OQ-1/2/3/6 and [test plan](testplan.md).

## Test Plan

### Unit Tests

Serializer tests cover two pool selections, credential omission and unchanged
legacy/generic provider fields. Validation rejects mismatched providers, malformed
profiles, unknown profiles, selector expressions and shared numeric IOPS.

### Integration Tests

Real PostgreSQL API/registered-gRPC tests verify persistence, masks, references,
immutability and zero outbound HTTPS requests. Operator Envtest checks current
configuration gates, retries, no Secret reads and class-before-backend cleanup.
The lab capture check uses real Kubernetes/AAP with catalog fixtures. AAP
receiver integration against native TBC/status fixtures remains onboarding work.

### E2E Tests

QE qualifies two-tenant encrypted FC VM I/O, authorized class/disk use and guarded
cleanup with shared-resource retention. No native FC E2E result is claimed here.

## Graduation Criteria

Developer Preview requires compatible receiver deployment, usable prepared shared
FC backend/pools, demonstrated VM disk use and guarded tenant cleanup. Capture-only
success establishes the sender boundary and is insufficient for release acceptance.

## Upgrade / Downgrade Strategy

Changes are additive; existing providers and ONTAP without profiles keep prior
behavior. Replace immutable bindings through normal dependency guards. Before
rollback to code lacking profiles, stop shared onboarding and resolve its OSAC-owned
configuration; shared customer infrastructure remains. Supported in-place OSAC
upgrades remain outside this feature's preview scope.

## Version Skew Strategy

New profiles require matched generated proto/API/operator/AAP revisions. Older
receivers expect prepared-SVM credentials and cannot consume this payload. Do not
activate shared profiles until the receiver is available. No CRD migration is added.

## Support Procedures

Inspect catalog separately from Tenant storage conditions, then the AAP job and
referenced TBC's native status. Array/TLS errors are diagnosed from Trident on the
hosting cluster. Pause storage reconciliation using existing deployment controls
before debugging; never delete the shared backend as a tenant-recovery shortcut.
Restoring the compatible receiver/native preparation permits normal retries.

## Infrastructure Needed

No new production service. Joint native acceptance needs qualified Trident,
FC-connected hosting workers, a prepared shared backend and VM/Volume workflow.

---

## Provenance

Authored: draft @ design 0.11.3 - 2bd6607, workspace osac-5813-netapp-integration @ c8d0d8890
Final: revise @ design 0.11.5 - 0c7ea35, workspace OSAC-5813-netapp-backend-tiers @ 4ebb5dd9c

> Context changed between draft and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"0c7ea35","source_repo":"4ebb5dd9c","source_repo_branch":"OSAC-5813-netapp-backend-tiers","commits_behind_main":0,"commits_ahead_main":18,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
