---
title: rework-tenant-storage-onboarding
authors:
  - zszabo@redhat.com
creation-date: 2026-06-08
last-updated: 2026-06-10
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-23
prd:
  - "prd.md"
see-also:
  - "/enhancements/tenant-storage-tiers"
replaces:
superseded-by:
---

# Rework Tenant Storage Onboarding

## Summary

Extract storage provisioning from the Tenant controller into a dedicated OSAC Storage Controller with a new TenantStorage API. The controller watches Tenant, ClusterOrder, and TenantStorage resources to manage the full storage lifecycle independently using a two-stage model. See [PRD](prd.md) for detailed requirements.

## Motivation

Storage provisioning logic — hub Secret checks, AAP job triggering, StorageClass resolution, and job tracking — currently runs inside the Tenant controller, and storage state (StorageClasses, jobs, conditions) lives in the Tenant CR. This couples storage lifecycle to tenant lifecycle: changes to storage onboarding risk regressions in namespace creation and UDN reconciliation. Supporting different storage workflows per delivery model (VMaaS vs CaaS) requires branching logic inside a single controller.

The extraction separates concerns: the same provisioning logic moves to a new controller, the same AAP provider interface is used, and the same status patterns apply. The design decisions center on where to draw the boundary between controllers and how the OSAC Storage Controller communicates with Tenant and ClusterOrder resources.

### Goals

- Reuse the existing `provisioning.RunProvisioningLifecycle()` framework and `ProvisioningProvider` interface without modifying the shared provisioning package. [Codebase: pkg/provisioning/provision_lifecycle.go]
- Follow the established controller pattern (Reconcile, handleUpdate/handleDelete, oldStatus compare, finalizer management).
- Implement Stage 1 and Stage 2 as separate operations to avoid coupling that would need to be untangled when CaaS support is added. [PRD: FR-6, FR-7]
- Name the controller broadly ("OSAC Storage Controller") to serve as the single entry point for all storage reconciliation. [PRD: Goal 3]

### Non-Goals

- Modifying the shared provisioning framework (pkg/provisioning/).
- Adding a feedback controller for TenantStorage (no fulfillment-service integration needed for this API).
- VAST-specific logic in the operator — the controller is provider-agnostic, VAST specifics live in AAP playbooks.
- StorageClass creation — the controller discovers existing SCs, it does not create them.
- CaaS Stage 2 trigger logic (OSAC-1123).

## Proposal

Introduce a TenantStorage API and an OSAC Storage Controller. The controller watches three resource types: Tenant (to create TenantStorage when a tenant reaches Ready and to handle teardown on deletion), TenantStorage (to drive the provisioning lifecycle), and ClusterOrder (for CaaS Stage 2 triggering). The AAP playbooks are split into four lifecycle actions matching the two-stage model.

The Tenant controller is simplified to manage only namespace creation and UDN reconciliation — it has zero storage responsibilities.

### Workflow Description

**Actor: Cloud Provider Admin (via Tenant CR creation)**

Starting state: A Tenant CR exists in the tenant namespace with Phase=Ready (namespace created).

**Stage 1 (Backend Setup):**

1. OSAC Storage Controller observes Tenant reaching Ready via Tenant watch.
2. Controller creates a TenantStorage resource in the tenant namespace with `spec.tenantRef` set to the Tenant name.
3. Controller reconciles TenantStorage: checks for hub Secret in osac-system namespace.
4. Hub Secret absent → controller triggers `osac-create-tenant-storage` AAP job. TenantStorage phase set to Progressing.
5. Controller polls AAP job status at StatusPollInterval (default 30s).
6. AAP job completes → hub Secret created by AAP with storage provider credentials.
7. Controller detects hub Secret via Secret watch → sets `StorageBackendReady` condition on both TenantStorage and Tenant CR. Stage 1 complete.

**Stage 2 (Cluster-Side Setup — VMaaS):**

8. For VMaaS, Stage 2 runs immediately after Stage 1. Controller discovers StorageClasses on the hub cluster by tenant label and populates the VMaaS cluster entry in TenantStorage status.
9. TenantStorage phase set to Ready.

**Stage 2 (Cluster-Side Setup — CaaS):**

- Controller watches ClusterOrder resources. When a ClusterOrder reaches Ready, a new CaaS cluster entry is added to TenantStorage status.
- CaaS Stage 2 trigger logic is covered by CaaS Tenant Storage Setup (OSAC-1123).

**Error handling:**

- AAP job fails → Phase set to Failed, job status recorded. Re-reconcile triggers backoff with retry.
- Hub Secret deleted after Ready → Secret watch triggers re-reconcile. StorageBackendReady set to False, re-triggers provisioning.

**Tenant deletion flow:**

1. OSAC Storage Controller observes Tenant deletion via Tenant watch.
2. Controller triggers `osac-delete-tenant-storage` AAP job (backend teardown).
3. Polls until terminal state. If teardown fails, BlockDeletionOnFailure prevents premature cleanup.
4. On success, controller deletes the TenantStorage resource.

### API Extensions

**New API: TenantStorage**

```go
type TenantStorageSpec struct {
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinLength=1
    TenantRef string `json:"tenantRef"`
}

type ClusterStorageEntry struct {
    ClusterName    string                 `json:"clusterName"`
    DeliveryModel  string                 `json:"deliveryModel"` // "VMaaS" or "CaaS"
    StorageReady   bool                   `json:"storageReady"`
    StorageClasses []ResolvedStorageClass  `json:"storageClasses,omitempty"`
}

type TenantStorageStatus struct {
    Phase          TenantStoragePhaseType  `json:"phase,omitempty"`
    ClusterEntries []ClusterStorageEntry   `json:"clusterEntries,omitempty"`
    Conditions     []metav1.Condition      `json:"conditions,omitempty"`
    Jobs           []JobStatus             `json:"jobs,omitempty"`
}
```

Phase values: Progressing, Ready, Failed, Deleting.
Condition types: `StorageBackendReady` (Stage 1 complete — hub Secret exists).
Print columns: Tenant (`spec.tenantRef`), Phase (`status.phase`).

Reused types: `ResolvedStorageClass` (from tenant_types.go), `JobStatus` (from job_types.go).

**Changes to Tenant API:** StorageClasses status field, Jobs status field, StorageClassReady condition type, and storage print columns removed. New `StorageBackendReady` condition type added, managed exclusively by the OSAC Storage Controller. [PRD: FR-2]

**Finalizer:** `osac.openshift.io/tenantstorage-finalizer` on TenantStorage resources.

**Provisioning lifecycle type switch:** `GetJobsFromResource()` in pkg/provisioning/provision_lifecycle.go adds a TenantStorage case and removes the Tenant case.

### Implementation Details/Notes/Constraints

**Controller watches:**

| Watch | Resource | Purpose |
|-------|----------|---------|
| Primary | TenantStorage | Standard controller reconciliation |
| Secondary | Tenant | Create TenantStorage when Ready, trigger teardown on deletion |
| Secondary | ClusterOrder | CaaS Stage 2 triggering (watch set up, trigger logic in OSAC-1123) |
| Secondary | Secrets (osac-system) | Re-reconcile when hub Secret appears/disappears |
| Secondary | StorageClasses | Re-reconcile when SCs with tenant label change |

**AAP job template mapping:**

| Job Template | Playbook | Triggered By | Stage |
|-------------|----------|-------------|-------|
| osac-create-tenant-storage | playbook_osac_create_tenant_storage.yml | Storage Controller | Stage 1 |
| osac-ensure-tenant-storage | playbook_osac_ensure_tenant_storage.yml | Storage Controller (VMaaS) / ClusterOrder (CaaS) | Stage 2 |
| osac-cleanup-tenant-storage | playbook_osac_cleanup_tenant_storage.yml | Storage Controller | Teardown |
| osac-delete-tenant-storage | playbook_osac_delete_tenant_storage.yml | Storage Controller | Teardown |

**AAP role changes:** The storage_provider dispatcher gains a `cleanup` action. The vast_storage role implements cleanup.yaml which removes Kubernetes resources by label, handling unreachable target clusters gracefully.

**ComputeInstance controller change:** Reads StorageClasses from the VMaaS cluster entry in TenantStorage status instead of Tenant status. Looks up TenantStorage by tenant name, checks TenantStoragePhaseReady. [PRD: FR-4]

**Stage 2 StorageClass discovery:** The controller queries StorageClasses on the target cluster using the `osac.openshift.io/tenant` label. For VMaaS, this is a local query on the hub cluster. For CaaS, this requires a remote client obtained from the ClusterOrder (future scope). Discovered SCs are recorded in the corresponding cluster entry in TenantStorage status.

### Security Considerations

The feature inherits the existing security model. Per-tenant hub Secrets remain in osac-system namespace with RBAC restricted to the operator service account. Admin credentials remain ephemeral via AAP pod env vars. [PRD: NFR-1, NFR-2]

The OSAC Storage Controller requires RBAC for: get/list/watch on Secrets, StorageClasses, Tenants, and ClusterOrders; full CRUD on TenantStorage resources; update on Tenant status (for StorageBackendReady condition).

### Failure Handling and Recovery

| Failure | Behavior | Recovery |
|---------|----------|----------|
| Hub Secret missing | Phase stays Progressing, AAP job triggered | Automatic retry via RunProvisioningLifecycle backoff |
| AAP job fails | Phase set to Failed, job status recorded | Re-reconcile triggers backoff with retry |
| AAP job times out | Poll continues at StatusPollInterval | Job eventually reaches terminal state |
| Hub Secret deleted after Ready | Secret watch triggers re-reconcile, StorageBackendReady set to False | Next reconcile re-triggers provisioning |
| Teardown job fails | Finalizer not removed, BlockDeletionOnFailure honored | Re-reconcile retries teardown |
| Controller restart mid-reconciliation | Job state persisted in TenantStorage status | Controller resumes polling on next reconcile |
| Controller down during Tenant deletion | TenantStorage not immediately deleted | Controller resumes on restart, detects orphaned TenantStorage |

Idempotency is guaranteed by the provisioning lifecycle framework — EvaluateAction() checks config version hashes to avoid duplicate triggers.

### RBAC / Tenancy

No RBAC or tenancy model changes. TenantStorage resources live in the tenant namespace, scoped by OSAC_TENANT_NAMESPACE. The `osac.openshift.io/tenant` annotation is set on the TenantStorage matching the parent Tenant name. StorageClass and Secret predicates filter by this annotation.

### Observability and Monitoring

No new observability changes. Existing monitoring mechanisms apply. TenantStorage status conditions and phase are observable via `kubectl get tenantstorage`. AAP job status is visible via `kubectl describe tenantstorage <name>`. The `StorageBackendReady` condition on the Tenant CR provides user-facing storage readiness visibility.

### Risks and Mitigations

**Breaking change in AAP playbook interface:** The previous combined playbook is replaced by four separate playbooks. Deploying AAP changes without operator changes breaks existing provisioning. Mitigation: coordinated PR merge ([osac-aap#338](https://github.com/osac-project/osac-aap/pull/338)), linked PR descriptions, deployment documentation. [PRD: NFR-3]

**Migration of existing tenants:** Existing tenants have storage state in the Tenant CR and no TenantStorage resource. The controller must create TenantStorage resources for existing tenants without re-triggering AAP provisioning. Mitigation: the controller detects existing tenants with hub Secrets already provisioned and creates TenantStorage resources with the correct state. [PRD: Risk 7.2]

**Controller-driven deletion complexity:** Without owner reference GC, the Storage Controller must handle the full Tenant deletion → TenantStorage teardown → TenantStorage deletion lifecycle explicitly. If the controller is down during Tenant deletion, TenantStorage resources could be orphaned. Mitigation: controller resumes on restart and watches for Tenants that no longer exist.

### Drawbacks

The OSAC Storage Controller adds a new controller to the operator, increasing operational complexity. Each storage-related reconciliation now involves two controllers (Tenant for namespace, Storage for storage) instead of one. This is justified by the decoupling benefit — the storage WG can iterate independently, and the two-stage model required for CaaS cannot be cleanly embedded in the Tenant controller.

Controller-driven deletion (watching Tenant events) is more complex than owner reference GC but provides explicit control over teardown ordering — AAP cleanup must complete before the TenantStorage resource is removed.

## Alternatives (Not Implemented)

### Keep storage logic in Tenant controller with feature flags

Keep all storage provisioning in the Tenant controller and use feature flags to enable/disable stages. Avoids a new API and controller but perpetuates coupling. Rejected because CaaS storage needs the ClusterOrder controller to trigger Stage 2 independently.

### Tenant controller creates TenantStorage

Have the Tenant controller create TenantStorage during onboarding with an owner reference. Simpler deletion model (GC handles it). Rejected because it keeps a storage responsibility in the Tenant controller. The Storage Controller should own the full storage lifecycle.

### ConfigMap instead of CRD

Use a ConfigMap per tenant for storage state. Avoids CRD registration but loses structured status, conditions, phase tracking, print columns, and kubectl integration.

### Flat TenantStorage status (no per-cluster entries)

Use a flat status with a single StorageClasses array. Simpler for VMaaS-only, but doesn't support CaaS where multiple clusters per tenant each have their own SCs. The per-cluster entries array is needed from v0.1 since CaaS is the primary delivery model.

## Open Questions

### 1. How does the OSAC Storage Controller determine the delivery model for Stage 2?

Proposed approach: VMaaS is implicit (every Ready Tenant gets a VMaaS cluster entry), CaaS is explicit (ClusterOrder triggers it). Event-driven, no configuration needed.

- **Owner:** Storage WG
- **Impact:** Stage 2 trigger logic, TenantStorage cluster entries population

### 2. What is the exact migration procedure for existing tenants?

Proposed approach: controller detects existing tenants with hub Secrets and creates TenantStorage with correct state. No reprovisioning triggered.

- **Owner:** Storage WG
- **Impact:** Controller startup behavior, upgrade procedure

## Test Plan

Unit tests cover the OSAC Storage Controller (TenantStorage creation on Tenant Ready, Stage 1 provisioning lifecycle, Stage 2 SC discovery, teardown on Tenant deletion, migration of existing tenants) and updated Tenant/ComputeInstance controllers. Integration testing uses envtest. E2E validation on the dev cluster before merge.

## Graduation Criteria

Graduation criteria will be defined when targeting a release. Expected stages: Dev Preview -> Tech Preview -> GA based on production deployment feedback.

## Upgrade / Downgrade Strategy

This introduces a new API (TenantStorage) and removes fields from Tenant API. On upgrade: new CRD is registered, operator creates TenantStorage resources for existing tenants on first reconcile (see Open Question 2). On downgrade: TenantStorage resources must be deleted before reverting, storage fields re-added to Tenant CRD.

AAP config-as-code and operator must be deployed together. See Risk section.

## Version Skew Strategy

The osac-operator and osac-aap must be updated together. Deploying the operator without updated AAP config-as-code causes the controller to fail finding new job templates — it retries with backoff until AAP is updated. Deploying AAP first breaks the existing Tenant controller.

## Support Procedures

**Failure detection:** `kubectl get tenantstorage` shows Phase (Failed indicates provisioning failure). `kubectl describe tenantstorage <name>` shows job history, cluster entries, and conditions. Controller logs show AAP job status transitions. `StorageBackendReady` condition on Tenant CR surfaces Stage 1 readiness to users.

**Disabling:** Set `osac.openshift.io/management-state: Unmanaged` annotation on individual TenantStorage resources. Or disable the controller entirely via OSAC_ENABLE_STORAGE_CONTROLLER=false.

**Recovery:** Remove the annotation or re-enable the controller. The controller re-evaluates state and resumes from where it left off. No data loss — TenantStorage status is the source of truth.

## Infrastructure Needed

None.
