# Test plan — OSAC-5813 shared Trident preview

## Overview

12 cases cover the same 5 local PRD anchors (R1–R5) and 6 interfaces (IC-1–IC-6)
defined in [design](design.md#interface-changes). Case IDs remain stable; the
expected behavior now follows the shared customer-managed TBC scope.

Unit, Envtest, component-integration and Contract belong to DEV. Deployed VM/disk
journeys belong to QE. A real API/database test does not establish native storage
support, and a capture job does not execute the receiving provider role.

## Planning evidence and execution paths

See [Integration testing](https://github.com/osac-project/osac/blob/main/docs/INTEGRATION-TESTING.md).

| Behavior / cases | Tier / owner | Suite / command | Real boundary and exclusions |
|---|---|---|---|
| API validation and persistence; R1-01, R2-01 | Unit with real PostgreSQL / configuration DEV | `go test ./internal/servers -run '^TestServers$' -timeout 10m` from fulfillment-service; registration suite `go test ./internal/cmd/service/start/grpcserver` | Actual handlers, registered gRPC, database and local HTTPS trap; array absent; no deployed CLI/service |
| Deployed CLI registration; R1-02 | Component integration / configuration DEV | Existing Fulfillment installer Kind suite; add NetApp registration case in `fulfillment-service/it/` | Planned real CLI/service/auth/database; no native provider lifecycle |
| Forms; R1-03 | Unit / UI team | Existing osac-ui form suites, once support is implemented | API hooks mocked; UI change deferred from this backend slice |
| Serialization; R2-02 | Unit / operator DEV | `go test ./pkg/provisioning` from osac-operator | Production serializer, fixture inputs; no AAP |
| Readiness/cleanup; R3-01, R4-01 subset | Envtest / operator DEV | `make test` from osac-operator; `storage_shared_trident_test.go` | Real API/etcd, public reconciliation; catalog and providers are doubles |
| Receiver setup/classes/cleanup; R3-02, R4-01 | Component integration / onboarding DEV | Proposed shared ONTAP target in `osac-aap/tests/integration/targets/`; existing `STORAGE_TESTS_ENABLED=true make test` | Real Ansible/Kind APIs; native TBC progression simulated; runner extension pending |
| Sender capture and native contract; R3-03 | Contract / configuration + onboarding DEV | Local lab `shared-capture.py` and current-package Go harness; native receiver runner pending | Capture: real Kubernetes/AAP, catalog fixtures, no native writes. Native receiving/Trident observation remains a gap under [OSAC-4843](https://redhat.atlassian.net/browse/OSAC-4843) and [OSAC-5813](https://redhat.atlassian.net/browse/OSAC-5813) |
| Two-tenant disks/retention; R4-02, R5-01/02 | E2E / QE with shared VM owner | Authorized manual FC lab acceptance; automate in `tests/e2e/` when live path exists | Real OSAC/AAP/Trident/ONTAP/CDI/KubeVirt/fabric; no fixture replacement qualifies |

## Test Cases

### R1: Register prepared backend configuration

#### TC-R1-01: Register locally without an array connection

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-1 | critical | automated | Unit / configuration DEV |

**Preconditions:** real handler/database and registered-gRPC harnesses; local
untrusted HTTPS connection trap; valid OSAC VALUE password reference metadata.

**Steps:** Create/Get a shared-profile backend, then update credentials/description
with masks. Supply malformed/duplicate/missing-mode profiles, mismatched provider,
invalid endpoint, missing endpoint/credentials and invalid/deleted Secret refs.
Attempt endpoint/profile mutation and test stale locking behavior.

**Expected:** valid configuration persists READY with exact TBC routing; zero
outbound connections and no protected password read. Invalid input is not saved.
Profile/endpoint changes are rejected, masked updates preserve unrelated fields,
and accepted updates clear historical ERROR/messages. Common endpoint/credential
requirements remain. **Executed:** handler and registered-gRPC coverage passes.

#### TC-R1-02: Deployed CLI registration persists both profile and selector

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-1, IC-3 | critical | automated | Component integration / configuration DEV |

**Preconditions:** matched API/CLI in the deployed Fulfillment Kind harness.

**Steps:** create a VALUE Secret, submit guide backend/tier files, Get/List both,
and attempt an immutable masked update; include an existing-provider backend.

**Expected:** persisted profile/TBC routing and pool selection match input;
READY means catalog acceptance even without array reachability. Public tier view
omits provider settings and existing-provider requests retain behavior.
**Status:** deployed new-schema CLI journey pending; in-process gRPC is separate evidence.

#### TC-R1-03: UI accepts shared NetApp configuration

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-3 | high | automated | Unit / UI DEV |

**Steps:** once forms are delivered, submit provider/profile/TBC fields and tier
pool selection; inject API validation failure and switch to VAST.

**Expected:** exact typed private fields, visible validation error and unchanged
VAST payload. No connectivity-success claim from catalog READY.
**Status:** UI-team follow-up, outside the current implementation slice.

### R2: Select prepared FC offerings

#### TC-R2-01: Reject incompatible shared-tier semantics

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-2 | critical | automated | Unit / configuration DEV |

**Steps:** register two tiers on one shared profile with different literal pool
labels. Try absent/unknown profile, empty/expression selector, nonzero numeric
IOPS/bandwidth, non-BLOCK, wrong provider and immutable binding updates. Repeat
legacy ONTAP caps and other-provider requests.

**Expected:** shared tiers retain exact profile/selectors and encryption boolean;
unsupported combinations fail before persistence. Legacy native caps and other
providers remain compatible. **Executed:** API suite passes.

#### TC-R2-02: Serialize references without unused credentials

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-4 | critical | automated | Unit / operator DEV |

**Steps:** resolve two tiers/one shared backend and serialize extra_vars; use
Secret clients that fail on reads. Also serialize VAST and a generic provider
connection with both common fields and provider config.

**Expected:** one connection with profiles; each tier has its own pool selector.
No shared-path endpoint/user/password/preparation, discovery read or preparation
lookup. Encryption remains metadata; existing/generic populated fields survive.
**Executed:** serializer and operator tests pass.

### R3: Keep setup and native readiness distinct

#### TC-R3-01: Gate readiness on current stages and classes

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-5, IC-6 | critical | automated | Envtest + Unit / operator and fulfillment DEV |

**Steps:** reconcile a Ready Tenant with two shared tiers, hold jobs pending or
failed, then complete setup/class jobs and publish matching classes. Change a
selector in a catalog fixture, provide unknown profile, and restart reconciliation.
Check Fulfillment's independent storage-condition projection.

**Expected:** no premature readiness or duplicate pending setup. Successful
current-config jobs plus exact classes establish storage readiness; stale success
cannot satisfy changed inputs. Unknown profile blocks dispatch. Missing/stale
observations remain unknown in API projection; identity Ready is separate.
**Execution:** shared lifecycle Envtest passes; fresh deployed feedback not exercised.

#### TC-R3-02: Receiver observes TBC and creates only tenant classes

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-4, IC-5, IC-6 | critical | automated | Component integration / onboarding DEV |

**Preconditions:** real Ansible/Kind, customer-owned FC TBC fixture, simulated
Trident status, native backendName different from TBC metadata.name.

**Steps:** run all setup/class actions twice. Try missing TBC/CRD, Unbound/Failed
state, wrong protocol, unavailable target and two backends with identical pool
labels. Exercise hub and a separate configured hosting target.

**Expected:** no direct ONTAP request or shared TBC/Secret mutation. Setup succeeds
only after native status is usable; classes select the exact bound backend and
literal pool labels. Native QoS/encryption defaults are untouched. Existing
provider outputs include class names; no cross-target fallback. **Status:** receiver pending.

#### TC-R3-03: Verify real operator → AAP delivery and native observation

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-4, IC-5, IC-6 | critical | manual | Contract / configuration + onboarding DEV |

**Steps:** first capture a shared two-tier request through production operator
packages, real Kubernetes and AAP. After compatible receiver deployment and
native authorization, repeat with real catalog records and observe existing TBC
status, resulting classes and feedback.

**Expected:** capture asserts references/selectors, no credential/preparation
reads, zero ONTAP requests/native writes, and intentionally fails before native
setup. The later native phase preserves all customer resources and publishes
only successful current-configuration readiness. **Executed subset:** lab capture job passed capture checks; catalog was a fixture, native receiver was not run.

### R4: Preserve shared resources during cleanup

#### TC-R4-01: Ordered, guarded and idempotent tenant cleanup

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-5, IC-6 | critical | automated | Envtest + component integration / owning DEV |

**Steps:** delete a Tenant after partial setup; hold/fail cleanup and restore it.
Check class-before-backend ordering with the public reconciler. Receiver tests
add another tenant/provider's classes, shared TBC/Secret, active dependencies and
UID mismatch, then replay teardown against hub/remote targets.

**Expected:** finalizer persists until ordered successful cleanup. Receiver removes
only fully owned departing-tenant configuration; no shared TBC/Secret/SVM/policy
mutation or deletion. Dependency/identity/target failures retain recoverable state.
**Executed subset:** operator deletion ordering/finalizer passes; native receiver pending.

#### TC-R4-02: Offboard without disrupting another tenant

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E / QE |

**Steps:** with two working tenant VMs on the shared TBC, attempt A's cleanup with
active data. Remove dependencies through the shared lifecycle, offboard A and
inspect retained resources and B's disk.

**Expected:** active data blocks cleanup. After authorized data removal, A's owned
classes/configuration disappear; shared TBC/native Secret/SVM/pools and B's disk
remain. B reads its saved value. **Status:** not run.

### R5: Consume authorized FC VM disks

#### TC-R5-01: Two tenants use the intended encrypted pools

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E / QE with OSAC-6037 owner |

**Preconditions:** authorized FC lab, qualified versions/workers/multipath,
customer-managed shared TBC/pools, matched receiver and shared VM lifecycle.

**Steps:** onboard two tenants/two tiers, provision VMs with DataVolumes through
the shared VM path, write distinct values, restart and read back. Record native
pool/QoS/encryption outcomes. Exercise hub hosting and dedicated hosting, one
selected target per deployment.

**Expected:** each disk uses its intended pool and advertised encryption, without
OSAC altering native policy. No duplicate allocation from OSAC bookkeeping. Actual
FC I/O succeeds from eligible hosting workers; identities/status/cleanup follow
OSAC-6037. Shared infrastructure is expected; per-tenant SVMs are not required.
**Status:** not run; capture-only delivery cannot satisfy this case.

#### TC-R5-02: Reject cross-tenant class and disk access

| Interfaces | Priority | Automation | Tier / owner |
|---|---|---|---|
| IC-5, IC-6 | critical | manual | E2E / QE |

**Steps:** on a shared FC worker, attempt A's request selecting B's tier/class,
native PVC/class access where available, native credential reads and guest access
to B's disk. Repeat from B toward A.

**Expected:** platform authorization rejects cross-tenant selection/Secret reads;
each guest receives only its authorized disk. A trusted worker may access both
LUNs, but guest disk isolation is enforced. Any cross-tenant access is a failure;
shared SVM/classes/pool labels alone do not prove isolation. **Status:** not run.

## Gaps

All five requirement anchors and six ICs retain planned cases. The shared sender
has automated API/serializer/Envtest coverage and a real Kubernetes/AAP capture.
Missing boundaries are deployed new-schema CLI registration, receiving Ansible
execution/native TBC observation, live feedback and all FC/VM/encryption/isolation/
shared-retention acceptance. UI remains separate team work. The receiving gap is
tracked under [OSAC-5813](https://redhat.atlassian.net/browse/OSAC-5813) and existing
[OSAC-4843](https://redhat.atlassian.net/browse/OSAC-4843) boundary work; shared VM
integration belongs to [OSAC-6037](https://redhat.atlassian.net/browse/OSAC-6037).
No unexecuted case is treated as passing because a fixture or capture passed.

## Summary

| Metric | Count |
|---|---|
| Total | 12 |
| Critical / High / Medium / Low | 11 / 1 / 0 / 0 |
| Automated / Manual | 8 / 4 |
| Requirements with cases | 5 / 5 |
| Interfaces with cases | 6 / 6 |
| Complete / partial / pending | 3 / 3 / 6 |
