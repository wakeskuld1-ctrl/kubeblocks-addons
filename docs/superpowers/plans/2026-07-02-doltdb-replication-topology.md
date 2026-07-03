# DoltDB Replication Topology Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement a creation-time DoltDB topology with one primary and one to five standbys.

**Architecture:** Keep the existing standalone ComponentDefinition and add one replication ComponentDefinition with replicasLimit 2 through 6. The startup wrapper renders per-Pod Dolt cluster config from KubeBlocks pod FQDN vars, and roleProbe publishes Dolt's SQL-reported role and epoch to KubeBlocks.

**Tech Stack:** Helm v3, KubeBlocks apps.kubeblocks.io/v1, DoltDB v2.1.10, POSIX shell, ShellCheck, kind-kbv12 runtime validation.

---

### Task 1: Add Replication Runtime Scripts

**Files:**
- Modify: `addons/doltdb/scripts/doltdb-start.sh`
- Create: `addons/doltdb/scripts/doltdb-role-probe.sh`
- Modify: `addons/doltdb/templates/scripts-template.yaml`

- [ ] Extend startup to copy the base config, append `cluster:` only when `DOLT_CLUSTER_MODE=true`, select bootstrap role from the current Pod ordinal, and remove optional database/user bootstrap env on standbys.
- [ ] Add roleProbe script that queries `@@GLOBAL.dolt_cluster_role` and `@@GLOBAL.dolt_cluster_role_epoch`, emits `primary <epoch>` or `standby <epoch>`, and fails on `detected_broken_config`.
- [ ] Mount the roleProbe script in the scripts ConfigMap.
- [ ] Run `bash -n addons/doltdb/scripts/doltdb-start.sh addons/doltdb/scripts/doltdb-role-probe.sh`.

### Task 2: Add Replication ComponentDefinitions

**Files:**
- Modify: `addons/doltdb/templates/_helpers.tpl`
- Modify: `addons/doltdb/templates/cmpd.yaml`
- Modify: `addons/doltdb/templates/clusterdefinition.yaml`
- Modify: `addons/doltdb/templates/cmpv.yaml`
- Modify: `addons/doltdb/templates/backuppolicytemplate.yaml`
- Modify: `addons/doltdb/values.yaml`

- [ ] Narrow the standalone ComponentDefinition regex so standalone matching does not include replication definitions.
- [ ] Render one replication ComponentDefinition with `replicasLimit.minReplicas=2` and `replicasLimit.maxReplicas=6`.
- [ ] Add `primary` and `standby` roles, roleProbe, primary-only SQL service routing, remotesapi port, pod FQDN vars, and cluster-mode env to replication definitions.
- [ ] Add `replication` topology to the ClusterDefinition.
- [ ] Include both standalone and replication definitions in ComponentVersion matching; keep replication backup/restore deferred.

### Task 3: Update Cluster Chart and Examples

**Files:**
- Modify: `addons-cluster/doltdb/values.yaml`
- Modify: `addons-cluster/doltdb/templates/cluster.yaml`
- Modify: `examples/doltdb/README.md`

- [ ] Add user-facing values `topology` and `replication.standbyReplicas`.
- [ ] Render standalone as exact `doltdb-<chartVersion>` with one replica.
- [ ] Render replication as exact `doltdb-replication-<chartVersion>` with total replicas `standbyReplicas + 1`.
- [ ] Document replication as creation-time only and Day-2 scale/switchover/failover as deferred.

### Task 4: Update Evidence and E2E

**Files:**
- Modify: `addons/doltdb/ENGINE.md`
- Modify: `hack/e2e-doltdb.sh`

- [ ] Update engine facts and capability status for the replication topology.
- [ ] Extend the e2e script with an opt-in replication case that writes through primary service, verifies standby read/readonly behavior, validates role labels, restarts, and checks data again.
- [ ] Keep backup/restore e2e on standalone unless a future replication restore case is proven.

### Task 5: Validate and Commit

**Files:**
- Verify changed files only.

- [ ] Run Helm lint for addon and cluster charts.
- [ ] Render standalone and replication topologies with `standbyReplicas=1` and `standbyReplicas=5`.
- [ ] Run shell syntax checks and ShellCheck severity error for DoltDB scripts and e2e.
- [ ] Run `git diff --check`.
- [ ] Run runtime replication smoke on `kind-kbv12` when static checks pass.
- [ ] Commit implementation with message `feat: add doltdb replication topology`.
