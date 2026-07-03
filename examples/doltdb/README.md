# DoltDB

DoltDB is a MySQL-compatible SQL database with Git-style versioning. This example shows how to run Dolt SQL Server on Kubernetes with KubeBlocks.

The addon exposes `standalone` and `replication` topologies. `replication` runs one primary and one to five standbys.

## Features In KubeBlocks

### Lifecycle Management

| Topology | Replicas | Horizontal scaling | Switchover | Failover | Vertical scaling | Expand volume | Restart | Stop/Start | Configure | Expose |
|----------|----------|-------------------|------------|----------|------------------|---------------|---------|------------|-----------|--------|
| standalone | 1 | No | No | No | Yes | Yes | Yes | Yes | Cluster API | Yes |
| replication | 1 primary + 1..5 standbys | Yes, standby scale-in/out | Yes, controlled | No | Yes | Yes | Yes | Yes | Cluster API | Primary SQL service |

### Backup and Restore

| Feature | Method | Description |
|---------|--------|-------------|
| Logical backup | dolt-backup | Lists current databases through SQL, stages each Dolt database repository on the target data volume, and syncs through `dolt_backup('sync-url', ...)` |

Restore from `dolt-backup` is supported for standalone and replication clusters through `Cluster.spec.restore`. KubeBlocks creates the DataProtection `Restore` CR automatically when the restored component reaches the post-ready stage. Replication backup targets the current primary; restore-post creates an empty Dolt commit per restored database to trigger standby catch-up without changing table rows.

The restore examples set `dataprotection.kubeblocks.io/source-target-name: doltdb` so KubeBlocks uses the recorded DoltDB backup source target directly when restoring into a different cluster name.

### Versions

| Major Versions | Description |
|---------------|-------------|
| 2.1 | 2.1.10 |

## Prerequisites

- Kubernetes cluster >= v1.21
- `kubectl` installed, refer to [K8s Install Tools](https://kubernetes.io/docs/tasks/tools/)
- Helm, refer to [Installing Helm](https://helm.sh/docs/intro/install/)
- KubeBlocks installed and running, refer to [Install Kubeblocks](../docs/prerequisites.md)
- DoltDB Addon enabled, refer to [Install Addons](../docs/install-addon.md)
- Create namespace `demo`:

  ```bash
  kubectl create ns demo
  ```

## Examples

### [Create](cluster.yaml)

Create a single-node DoltDB cluster with an initial database named `testdb`:

```bash
kubectl apply -f examples/doltdb/cluster.yaml
```

Check cluster and pod status:

```bash
kubectl get -n demo cluster doltdb-cluster
kubectl get pod -n demo -l app.kubernetes.io/instance=doltdb-cluster,apps.kubeblocks.io/component-name=doltdb
```

Connect with the Dolt CLI from a temporary client Pod:

```bash
ROOT_PASSWORD="$(kubectl get secret -n demo doltdb-cluster-doltdb-account-root -o jsonpath='{.data.password}' | base64 --decode)"
SERVICE="$(kubectl get svc -n demo -l app.kubernetes.io/instance=doltdb-cluster,apps.kubeblocks.io/component-name=doltdb -o jsonpath='{.items[?(@.spec.clusterIP!="None")].metadata.name}')"

kubectl run -n demo doltdb-client --rm -it --restart=Never \
  --image=docker.io/dolthub/dolt-sql-server:2.1.10 \
  -- dolt --host="${SERVICE}.demo.svc" --port=3306 --user=root \
  --password="${ROOT_PASSWORD}" --use-db=testdb --no-tls \
  sql --query="SHOW TABLES;" --result-format=csv
```

When TLS is enabled, omit `--no-tls`.

### [Create replication topology](cluster-replication.yaml)

Create a DoltDB cluster with one primary and one standby:

```bash
kubectl apply -f examples/doltdb/cluster-replication.yaml
```

Check Dolt roles reported by KubeBlocks:

```bash
kubectl get pod -n demo \
  -l app.kubernetes.io/instance=doltdb-replication,apps.kubeblocks.io/component-name=doltdb \
  --show-labels
```

The default SQL service routes to the `primary` role. Standbys are read-only and receive committed writes from the primary. Controlled switchover is supported. Automatic failover and failback are intentionally not supported.

### Monitoring

Dolt native Prometheus metrics are enabled on the `metrics` port (`11228`) at `/metrics`. You can verify the endpoint directly from the main container without a local Prometheus or PodMonitor:

```bash
POD="$(kubectl get pod -n demo -l app.kubernetes.io/instance=doltdb-cluster,apps.kubeblocks.io/component-name=doltdb --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
kubectl exec -n demo "$POD" -c doltdb -- /usr/bin/curl -fsS http://127.0.0.1:11228/metrics \
  | grep -E 'dss_(dolt_version|concurrent_connections|is_replica|replication_lag)'
```

Replication clusters should expose `dss_is_replica` and `dss_replication_lag` after data has been written and replicated. The addon does not ship a Grafana dashboard because no official Dolt dashboard was found. If you import a custom dashboard, build variables from the actual scraped labels, especially KubeBlocks pod labels and Dolt's `namespace`, `cluster`, and `component` metric labels.

### [Scale out replication](scale-out.yaml)

Add one standby to a replication cluster:

```bash
kubectl apply -f examples/doltdb/scale-out.yaml
```

The addon renders replication peer variables into the config template so KubeBlocks restarts the component after the peer list changes. After restart, every Pod regenerates its Dolt `cluster.standby_remotes` list from the current component Pod FQDN list.

### [Scale in specified standby](scale-in-specified-standby.yaml)

Scale-in must target a standby explicitly:

```bash
kubectl apply -f examples/doltdb/scale-in-specified-standby.yaml
```

Before applying, set `onlineInstancesToOffline` to a current standby Pod name. Do not target the current primary. Use the role labels or `/scripts/doltdb-role-probe.sh` to confirm the target role first.

### [Switchover replication](switchover.yaml)

Controlled switchover transfers the primary role to a selected standby. Automatic failover and failback are not supported.

Check the current role labels and edit `instanceName` / `candidateName` in `switchover.yaml` before applying:

```bash
kubectl get pod -n demo \
  -l app.kubernetes.io/instance=doltdb-replication,apps.kubeblocks.io/component-name=doltdb \
  -L kubeblocks.io/role

kubectl apply -f examples/doltdb/switchover.yaml
kubectl wait -n demo --for=jsonpath='{.status.phase}'=Succeed \
  opsrequest/doltdb-replication-switchover --timeout=300s
```

After a switchover, `doltdb-replication-doltdb-0` may be a standby. Restarts preserve Dolt's persisted role state instead of recalculating roles from ordinal.

### [Restart replication](restart-replication.yaml)

Restart all replication Pods, for example after a controlled switchover:

```bash
kubectl apply -f examples/doltdb/restart-replication.yaml
kubectl wait -n demo --for=jsonpath='{.status.phase}'=Succeed \
  opsrequest/doltdb-replication-restart --timeout=300s
```

### [Configure through Cluster API](cluster-with-config.yaml)

DoltDB server settings are rendered into the pod-local server config from `spec.componentSpecs[].configs[].variables`. Changes restart the component.

```bash
kubectl apply -f examples/doltdb/cluster-with-config.yaml
```

To update configuration later, edit the Cluster variables and re-apply. For example, change `DOLT_LOG_LEVEL` from `debug` to `warning`:

```yaml
configs:
  - name: doltdb-server-config
    restart: true
    variables:
      DOLT_LOG_LEVEL: warning
```

> [!IMPORTANT]
> `read_timeout_millis` and `write_timeout_millis` are stored in milliseconds but applied as second-based SQL timeouts. Use whole-second values such as `12000`, not values like `12345`.

OpsRequest-based reconfiguration is not supported in the current addon version.

### [Create with TLS](cluster-tls.yaml)

Enable KubeBlocks-managed TLS for the SQL listener:

```bash
kubectl apply -f examples/doltdb/cluster-tls.yaml
```

### [Vertical scaling](verticalscale.yaml)

```bash
kubectl apply -f examples/doltdb/verticalscale.yaml
```

### [Expand volume](volumeexpand.yaml)

Make sure your StorageClass supports volume expansion before applying this example.

```bash
kubectl apply -f examples/doltdb/volumeexpand.yaml
```

### [Restart](restart.yaml)

```bash
kubectl apply -f examples/doltdb/restart.yaml
```

### [Stop](stop.yaml)

```bash
kubectl apply -f examples/doltdb/stop.yaml
```

### [Start](start.yaml)

```bash
kubectl apply -f examples/doltdb/start.yaml
```

### [Backup standalone](backup.yaml)

> [!IMPORTANT]
> Create a `BackupRepo` before running backups. Refer to [BackupRepo](../docs/create-backuprepo.md).

Write and commit sample data on the standalone source cluster before creating a backup:

```bash
SOURCE_CLUSTER=doltdb-cluster
CLIENT_IMAGE=docker.io/dolthub/dolt-sql-server:2.1.10
ROOT_PASSWORD="$(kubectl get secret -n demo "${SOURCE_CLUSTER}-doltdb-account-root" -o jsonpath='{.data.password}' | base64 --decode)"
SERVICE="$(kubectl get svc -n demo -l app.kubernetes.io/instance="${SOURCE_CLUSTER}",apps.kubeblocks.io/component-name=doltdb -o jsonpath='{.items[?(@.spec.clusterIP!="None")].metadata.name}')"

run_sql() {
  local query="$1"
  kubectl run -n demo doltdb-client --rm -i --restart=Never --image="${CLIENT_IMAGE}" -- \
    dolt --host="${SERVICE}.demo.svc" --port=3306 --user=root \
    --password="${ROOT_PASSWORD}" --use-db=testdb --no-tls \
    sql "--query=${query}" --result-format=csv
}

run_sql "CREATE TABLE IF NOT EXISTS kb_smoke (id int primary key, note varchar(64));"
run_sql "REPLACE INTO kb_smoke VALUES (1, 'standalone-ok');"
run_sql "CALL DOLT_ADD('-A');"
run_sql "CALL DOLT_COMMIT('-m', 'example standalone backup');"
```

Create the backup:

```bash
kubectl apply -f examples/doltdb/backup.yaml
kubectl wait -n demo --for=jsonpath='{.status.phase}'=Completed backup/doltdb-cluster-backup --timeout=300s
```

Supported backup methods on the cluster BackupPolicy:

```bash
kubectl get backuppolicy -n demo doltdb-cluster-doltdb-backup-policy -oyaml | yq '.spec.backupMethods[].name'
```

Expected methods:

```text
dolt-backup
```

### [Backup replication](backup-replication.yaml)

Replication backup uses the current primary target from the generated BackupPolicy. The generated BackupPolicy name still follows `<cluster-name>-doltdb-backup-policy`.

Seed data in the default database and a second database so the backup covers every current Dolt database:

```bash
SOURCE_CLUSTER=doltdb-replication
CLIENT_IMAGE=docker.io/dolthub/dolt-sql-server:2.1.10
ROOT_PASSWORD="$(kubectl get secret -n demo "${SOURCE_CLUSTER}-doltdb-account-root" -o jsonpath='{.data.password}' | base64 --decode)"
SERVICE="$(kubectl get svc -n demo -l app.kubernetes.io/instance="${SOURCE_CLUSTER}",apps.kubeblocks.io/component-name=doltdb -o jsonpath='{.items[?(@.spec.clusterIP!="None")].metadata.name}')"

run_sql() {
  local query="$1"
  local database="${2:-testdb}"
  local db_arg=()
  if [ -n "$database" ]; then
    db_arg=(--use-db="${database}")
  fi
  kubectl run -n demo doltdb-client --rm -i --restart=Never --image="${CLIENT_IMAGE}" -- \
    dolt --host="${SERVICE}.demo.svc" --port=3306 --user=root \
    --password="${ROOT_PASSWORD}" "${db_arg[@]}" --no-tls \
    sql "--query=${query}" --result-format=csv
}

run_sql "CREATE TABLE IF NOT EXISTS kb_smoke (id int primary key, note varchar(64));"
run_sql "REPLACE INTO kb_smoke VALUES (1, 'replication-ok');"
run_sql "CALL DOLT_ADD('-A');"
run_sql "CALL DOLT_COMMIT('-m', 'example replication backup');"

run_sql "CREATE DATABASE IF NOT EXISTS auditdb;" ""
run_sql "CREATE TABLE IF NOT EXISTS kb_smoke (id int primary key, note varchar(64));" auditdb
run_sql "REPLACE INTO kb_smoke VALUES (10, 'audit-ok');" auditdb
run_sql "CALL DOLT_ADD('-A');" auditdb
run_sql "CALL DOLT_COMMIT('-m', 'example audit backup');" auditdb
```

Create the replication backup:

```bash
kubectl apply -f examples/doltdb/backup-replication.yaml
kubectl wait -n demo --for=jsonpath='{.status.phase}'=Completed \
  backup/doltdb-replication-backup --timeout=300s
```

### [Restore standalone](restore.yaml)

Create a new one-replica target cluster from the completed backup, then wait for the automatic post-ready restore action:

```bash
kubectl apply -f examples/doltdb/restore.yaml

kubectl wait -n demo --for=condition=Ready pod \
  -l app.kubernetes.io/instance=doltdb-cluster-restore,apps.kubeblocks.io/component-name=doltdb --timeout=300s

RESTORE="$(kubectl get restore -n demo \
  -l dataprotection.kubeblocks.io/backup-name=doltdb-cluster-backup,app.kubernetes.io/instance=doltdb-cluster-restore \
  -o go-template='{{range .items}}{{if .spec.readyConfig}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | head -n 1)"
kubectl wait -n demo --for=jsonpath='{.status.phase}'=Completed "restore/${RESTORE}" --timeout=300s
```

Verify the restored data through a remote Dolt client:

```bash
RESTORE_CLUSTER=doltdb-cluster-restore
CLIENT_IMAGE=docker.io/dolthub/dolt-sql-server:2.1.10
ROOT_PASSWORD="$(kubectl get secret -n demo "${RESTORE_CLUSTER}-doltdb-account-root" -o jsonpath='{.data.password}' | base64 --decode)"
SERVICE="$(kubectl get svc -n demo -l app.kubernetes.io/instance="${RESTORE_CLUSTER}",apps.kubeblocks.io/component-name=doltdb -o jsonpath='{.items[?(@.spec.clusterIP!="None")].metadata.name}')"

kubectl run -n demo doltdb-client --rm -i --restart=Never --image="${CLIENT_IMAGE}" -- \
  dolt --host="${SERVICE}.demo.svc" --port=3306 --user=root \
  --password="${ROOT_PASSWORD}" --use-db=testdb --no-tls \
  sql --query="SELECT id,note FROM kb_smoke ORDER BY id;" --result-format=csv
```

### [Restore replication](restore-replication.yaml)

Create a new primary/standby target cluster from the completed backup, then wait for the automatic post-ready restore action:

```bash
kubectl apply -f examples/doltdb/restore-replication.yaml

kubectl wait -n demo --for=condition=Ready pod \
  -l app.kubernetes.io/instance=doltdb-replication-restore,apps.kubeblocks.io/component-name=doltdb --timeout=300s

RESTORE="$(kubectl get restore -n demo \
  -l dataprotection.kubeblocks.io/backup-name=doltdb-replication-backup,app.kubernetes.io/instance=doltdb-replication-restore \
  -o go-template='{{range .items}}{{if .spec.readyConfig}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | head -n 1)"
kubectl wait -n demo --for=jsonpath='{.status.phase}'=Completed "restore/${RESTORE}" --timeout=300s
```

KubeBlocks selects the restored primary for post-ready restore from the component role labels. After restoring each database, the addon creates an empty Dolt commit on the restored primary to trigger standby catch-up.

Verify restored data on every restored Pod through a remote Dolt client:

```bash
RESTORE_CLUSTER=doltdb-replication-restore
CLIENT_IMAGE=docker.io/dolthub/dolt-sql-server:2.1.10
ROOT_PASSWORD="$(kubectl get secret -n demo "${RESTORE_CLUSTER}-doltdb-account-root" -o jsonpath='{.data.password}' | base64 --decode)"

for POD in $(kubectl get pod -n demo \
  -l app.kubernetes.io/instance="${RESTORE_CLUSTER}",apps.kubeblocks.io/component-name=doltdb \
  --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); do
  HOST="$(kubectl get pod -n demo "${POD}" -o jsonpath='{.status.podIP}')"
  kubectl run -n demo doltdb-client --rm -i --restart=Never --image="${CLIENT_IMAGE}" -- \
    dolt --host="${HOST}" --port=3306 --user=root \
    --password="${ROOT_PASSWORD}" --use-db=testdb --no-tls \
    sql --query="SELECT id,note FROM kb_smoke ORDER BY id;" --result-format=csv

  kubectl run -n demo doltdb-client --rm -i --restart=Never --image="${CLIENT_IMAGE}" -- \
    dolt --host="${HOST}" --port=3306 --user=root \
    --password="${ROOT_PASSWORD}" --use-db=auditdb --no-tls \
    sql --query="SELECT id,note FROM kb_smoke ORDER BY id;" --result-format=csv
done
```

## Delete

To delete the cluster and its PVCs:

```bash
kubectl delete cluster -n demo doltdb-cluster
kubectl delete cluster -n demo doltdb-cluster-restore
kubectl delete cluster -n demo doltdb-replication
kubectl delete cluster -n demo doltdb-replication-restore
```

To also remove backup data from the repository when deleting the Backup CR, keep `deletionPolicy: Delete` in `backup.yaml` and `backup-replication.yaml`.
