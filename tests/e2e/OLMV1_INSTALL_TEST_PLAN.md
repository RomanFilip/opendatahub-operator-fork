# OLMv1 ClusterExtension Install Helpers — Testing Plan

**Branch:** `RHOAIENG-71042-olmv1-e2e-infra`
**Jira:** RHOAIENG-71042

## Scope

Tests for the three new helpers in `test_context_test.go` and the dispatch field in `helper_test.go`:

| Symbol | File | Testable how |
|--------|------|--------------|
| `EnsureOperatorInstalledViaClusterExtension` | `test_context_test.go:1086` | E2E only |
| `ensureClusterExtensionSAExists` | `test_context_test.go:1095` | E2E only |
| `ensureClusterExtensionInstalled` | `test_context_test.go:1117` | E2E only |
| `ensureClusterExtensionReady` | `test_context_test.go:1152` | E2E only |
| `Operator.useOLMv1` dispatch | `helper_test.go:548` | E2E only |

All helpers are `*TestContext` methods. They call `EventuallyResourceCreatedOrUpdated`,
`EnsureResourceExists`, and a real `client.Client` — no unit testing without a running
API server. `envtest` would work but is not worth the setup for dispatch-only logic.

---

## Test Scenarios

### 1. Happy Path — full install via ClusterExtension (E2E, needs OLMv1 cluster)

**Goal:** `EnsureOperatorInstalledViaClusterExtension` succeeds end-to-end.

**Pre-conditions:** OCP 4.18+ cluster with `clusterextensions.olm.operatorframework.io` CRD
and `openshift-redhat-operators` ClusterCatalog.

**Trigger:** tag one operator in `creation_test.go` with `useOLMv1: true` (Remaining Work
item from the implementation plan).

**Assertions (independent cluster signals, not production-code reads):**

```bash
# SA created in install namespace with correct name
kubectl get sa olmv1-installer-<package> -n <namespace>

# CRB exists, bound to cluster-admin
kubectl get clusterrolebinding olmv1-installer-<package> -o json \
  | jq '.roleRef.name == "cluster-admin"'

# ClusterExtension exists with correct catalog selector
kubectl get clusterextension <package> -o json \
  | jq '.spec.source.catalog.selector.matchLabels
        ["olm.operatorframework.io/metadata.name"] == "openshift-redhat-operators"'

# Installed condition is True
kubectl get clusterextension <package> -o json \
  | jq '.status.conditions[] | select(.type=="Installed") | .status == "True"'
```

**Run command:**
```bash
make e2e-test -run TestCreation/ValidateOperatorsInstallation
```

---

### 2. Idempotency — second call on existing ClusterExtension (E2E)

**Goal:** calling `ensureClusterExtensionInstalled` when the `ClusterExtension` already
exists returns without error (the fetch-first guard at `test_context_test.go:1118`).

**How:** run `ValidateOperatorsInstallation` twice consecutively with the same operator
tagged `useOLMv1: true`. The second run must not fail with an immutable-field CEL error.

**Failure signal without this test:** `ensureClusterExtensionInstalled` tries to apply
`EventuallyResourceCreatedOrUpdated` on an existing resource → the API rejects the
immutable `spec.namespace` / `spec.serviceAccount.name` fields.

---

### 3. OLMv0 regression (E2E, must stay green)

**Goal:** operators that do NOT set `useOLMv1: true` still install correctly via OLMv0.

**Run command:**
```bash
make e2e-test -run TestCreation/ValidateOperatorsInstallation
```

All existing operators (kueue, leaderWorkerSet, jobSet while they still have
`useOLMv1: false`) must reach `InstallSucceeded` through the OLMv0 path.
The new `if op.useOLMv1` branch at `helper_test.go:548` must not touch them.

---

### 4. SA / CRB naming convention (manual, no cluster needed)

**Goal:** verify no copy-paste drift in the three places the name `olmv1-installer-<package>` is assembled.

```bash
grep -n '"olmv1-installer-"' tests/e2e/test_context_test.go
```

Expected: exactly **three** occurrences — SA name (~line 1099), CRB name (~line 1107),
and serviceAccount ref in the ClusterExtension spec (~line 1131). A fourth occurrence
means drift.

---

### 5. GVK constant reachability (build-time)

```bash
make build
```

`gvk.ClusterExtension` is used in `test_context_test.go`. A build failure here means the
constant is missing or the import path changed.

---

## Known Gap: No Cleanup Registered

`ensureClusterExtensionSAExists` and `ensureClusterExtensionInstalled` do **not** call
`registerCleanup`. SA, CRB, and ClusterExtension persist after the test completes.

**Current state:** acceptable — these are cluster-setup helpers, not per-test resources.
ClusterExtensions are cluster-scoped; leaving them avoids repeated install/uninstall churn
across test runs.

**Risk:** a stale ClusterExtension with wrong channel or namespace from a previous run will
be silently reused (fetch-first returns early). If the spec needs to change, the resource
must be manually deleted before re-running.

**Upgrade path:** add `registerCleanup` calls inside `ensureClusterExtensionInstalled` and
`ensureClusterExtensionSAExists` if tests show cross-run contamination.

---

## Execution Order

1. `make build` — verifies GVK constant (no cluster needed)
2. Manual grep for naming drift (no cluster needed)
3. Pre-flight cluster check:
   ```bash
   kubectl get clustercatalog openshift-redhat-operators
   kubectl get crd clusterextensions.olm.operatorframework.io
   ```
4. Tag one operator `useOLMv1: true` in `creation_test.go` (Remaining Work)
5. Run `TestCreation/ValidateOperatorsInstallation` — verifies happy path + OLMv0 regression
6. Run same test again — verifies idempotency

---

## Remaining Work (prerequisite for scenarios 1 and 2)

- [ ] Confirm `leaderWorkerSet` and `jobSet` packages exist in `openshift-redhat-operators`
      catalog:
      ```bash
      kubectl get clustercatalog openshift-redhat-operators -o json | jq '.status'
      ```
- [ ] Tag one operator in `creation_test.go` with `useOLMv1: true`
- [ ] Run scenario 1 to validate end-to-end

Scenario 3 (OLMv0 regression) and scenarios 4–5 (static checks) run without any remaining work.
