# DoltDB Replication Topology Design

## Goal

Add a creation-time DoltDB replication topology for KubeBlocks with one primary and one to five standbys, while preserving the existing one-replica standalone topology.

## Scope

This phase implements Provision for the `doltdb` v2.1.10 replication topology on the live KubeBlocks v1.2 API exposed by `kind-kbv12`.

In scope:

- `topology=replication`.
- Total replicas `2..6`, expressed in the cluster chart as `replication.standbyReplicas: 1..5`.
- Pod-identity based Dolt `cluster` config generation at startup.
- KubeBlocks roles `primary` and `standby` from Dolt SQL system variables.
- Primary-only SQL service routing.
- Static and runtime validation for the smallest replication topology.

Out of scope:

- Day-2 scale-out and scale-in.
- Switchover, automatic failover, and failback.
- Multi-node restore support; it remains deferred until a targeted runtime e2e proves restore into the replication topology.
- Parameter management, observability, TLS runtime validation, and pod security hardening.

## Engine Facts

Dolt direct-to-standby replication is configured in SQL server YAML under `cluster.standby_remotes`, `cluster.bootstrap_role`, `cluster.bootstrap_epoch`, and `cluster.remotesapi.port`. Each server must point its standby remotes at the other servers' remotesapi endpoints. Bootstrap role and epoch only apply to a newly run server; persisted role and epoch override bootstrap config on restart.

Dolt exposes its authoritative cluster role and epoch through `@@GLOBAL.dolt_cluster_role` and `@@GLOBAL.dolt_cluster_role_epoch`. Supported runtime roles are `primary`, `standby`, and `detected_broken_config`; `detected_broken_config` represents a cluster misconfiguration and must not be published as a healthy KubeBlocks role.

Dolt validates that a `cluster` stanza contains at least one `standby_remote`, so the existing standalone topology must not be forced through cluster mode.

## KubeBlocks Mapping

The addon keeps the existing standalone `ComponentDefinition` fixed at one replica. It adds one replication `ComponentDefinition` with `replicasLimit.minReplicas=2` and `replicasLimit.maxReplicas=6`. The cluster chart still validates `replication.standbyReplicas: 1..5` and renders the selected creation-time replica count.

The cluster chart renders:

- `topology: standalone`, `componentDef: doltdb`, `replicas: 1` for standalone.
- `topology: replication`, `componentDef: doltdb-replication-<chartVersion>`, `replicas: <total>` for replication.

The runtime uses KubeBlocks `componentVarRef.podFQDNs` and the current Pod name from the downward API to generate a per-Pod cluster config file before delegating to the official Dolt SQL server entrypoint.

## Validation

Static validation will render and inspect both standalone and replication forms. Runtime validation on `kind-kbv12` will create the smallest replication topology, wait for roles, write through the primary service, verify reads from primary and standby, verify standby write rejection, restart the component, and verify data remains readable.
