# CloudNativePG (CNPG) Helm Chart

A production-ready Helm chart for deploying PostgreSQL clusters using CloudNativePG operator on Kubernetes.

## Overview

This Helm chart simplifies the deployment of highly available PostgreSQL clusters managed by the CloudNativePG operator. It provides automated failover, backup/recovery capabilities, and enterprise-grade PostgreSQL management on Kubernetes.

## Features

- ✅ CloudNativePG operator-based PostgreSQL cluster management
- ✅ High availability with automatic failover
- ✅ Persistent storage with configurable storage classes
- ✅ Optimized PostgreSQL parameters for production workloads
- ✅ Built-in security with superuser and application user management
- ✅ Host-based authentication (pg_hba) configuration
- ✅ Post-initialization SQL script support
- ✅ Rolling updates with configurable strategies
- ✅ PostgreSQL 15.x support

## Prerequisites

- Kubernetes 1.23+
- Helm 3.0+
- **CloudNativePG Operator installed** (Required!)
- PersistentVolume provisioner support
- StorageClass configured (default: Longhorn)

## Installing CloudNativePG Operator

Before deploying this chart, you must install the CloudNativePG operator:

```bash
# Install the operator
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.0.yaml

# Verify the operator is running
kubectl get deployment -n cnpg-system cnpg-controller-manager
```

Or using Helm:

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg
```

## Installation

### Basic Installation

```bash
# Create namespace
kubectl create namespace pg

# Install the chart
helm install my-postgres ./postgres-cnpg
```

### Install with Custom Values

```bash
helm install my-postgres ./postgres-cnpg -f custom-values.yaml
```

### Install in Existing Namespace

```bash
helm install my-postgres ./postgres-cnpg \
  --set cluster.namespace=database \
  --create-namespace
```

## Configuration

### Cluster Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cluster.name` | Name of the PostgreSQL cluster | `my-pgsql-cluster` |
| `cluster.namespace` | Namespace for the cluster | `pg` |
| `cluster.description` | Cluster description | `My example pg cluster` |
| `cluster.image` | PostgreSQL container image | `ghcr.io/cloudnative-pg/postgresql:15.1` |
| `cluster.instances` | Number of PostgreSQL instances | `2` |

### Authentication Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cluster.superuserSecretName` | Name of superuser secret | `pg-superuser` |
| `cluster.enableSuperuserAccess` | Enable superuser access | `true` |

> **Security Warning:** Change default passwords in `secrets.yaml` before production deployment!

### Lifecycle Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cluster.startDelay` | Delay before starting (seconds) | `30` |
| `cluster.stopDelay` | Delay before stopping (seconds) | `100` |
| `cluster.primaryUpdateStrategy` | Update strategy for primary | `unsupervised` |

### PostgreSQL Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cluster.postgresql.parameters.max_connections` | Maximum number of connections | `200` |
| `cluster.postgresql.parameters.shared_buffers` | Shared memory buffers | `256MB` |
| `cluster.postgresql.parameters.effective_cache_size` | Effective cache size | `768MB` |
| `cluster.postgresql.parameters.maintenance_work_mem` | Maintenance work memory | `64MB` |
| `cluster.postgresql.parameters.work_mem` | Work memory per operation | `655kB` |
| `cluster.postgresql.parameters.max_wal_size` | Maximum WAL size | `4GB` |
| `cluster.postgresql.pg_hba` | Host-based authentication rules | `[host all all 10.240.0.0/16 scram-sha-256]` |

### Bootstrap Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cluster.bootstrap.initdb.database` | Initial database name | `my_app_db` |
| `cluster.bootstrap.initdb.owner` | Database owner | `app_user` |
| `cluster.bootstrap.initdb.secretName` | Secret for app user credentials | `pg-app-user` |
| `cluster.bootstrap.initdb.postInitApplicationSQL` | SQL scripts to run after init | `[create schema my_app]` |

### Storage Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cluster.storage.size` | Storage size per instance | `1Gi` |
| `cluster.storage.storageClass` | StorageClass name | `longhorn` |

## Default Credentials

The chart creates two secrets with default credentials:

### Superuser (postgres)
- **Username:** `postgres`
- **Password:** `superuser-password`
- **Secret Name:** `pg-superuser`

### Application User
- **Username:** `app_user`
- **Password:** `app-user-password`
- **Secret Name:** `pg-app-user`

> **⚠️ CRITICAL:** These are example credentials. You MUST change them before deploying to production!

## Usage Examples

### Production Cluster with 3 Replicas

```bash
helm install prod-postgres ./postgres-cnpg \
  --set cluster.name=prod-pgsql \
  --set cluster.instances=3 \
  --set cluster.storage.size=50Gi \
  --set cluster.postgresql.parameters.shared_buffers=2GB \
  --set cluster.postgresql.parameters.effective_cache_size=6GB
```

### Development Cluster

```bash
helm install dev-postgres ./postgres-cnpg \
  --set cluster.name=dev-pgsql \
  --set cluster.instances=1 \
  --set cluster.storage.size=5Gi
```

### Custom Database and Schema

```bash
helm install app-postgres ./postgres-cnpg \
  --set cluster.bootstrap.initdb.database=myapp_db \
  --set cluster.bootstrap.initdb.owner=myapp_user \
  --set cluster.bootstrap.initdb.postInitApplicationSQL[0]="create schema myapp"
```

## Accessing PostgreSQL

### Get Connection Information

```bash
# Get the cluster status
kubectl get cluster -n pg

# Get the service endpoints
kubectl get svc -n pg
```

### Connect from Within Kubernetes

The cluster creates several services:

- **Primary service (read-write):** `<cluster-name>-rw`
- **Replica service (read-only):** `<cluster-name>-ro`
- **Any instance service:** `<cluster-name>-r`

Connection string example:
```
postgresql://app_user:app-user-password@my-pgsql-cluster-rw.pg.svc.cluster.local:5432/my_app_db
```

### Port Forward for Local Access

```bash
kubectl port-forward -n pg svc/my-pgsql-cluster-rw 5432:5432
```

Then connect using:
```bash
psql -h localhost -U app_user -d my_app_db
```

### Using psql from a Pod

```bash
kubectl run -it --rm psql \
  --image=postgres:15 \
  --restart=Never \
  -n pg \
  -- psql -h my-pgsql-cluster-rw -U app_user -d my_app_db
```

## Cluster Management

### Check Cluster Status

```bash
kubectl get cluster -n pg
kubectl describe cluster my-pgsql-cluster -n pg
```

### View Cluster Pods

```bash
kubectl get pods -n pg -l cnpg.io/cluster=my-pgsql-cluster
```

### Check Primary Instance

```bash
kubectl get cluster my-pgsql-cluster -n pg -o jsonpath='{.status.currentPrimary}'
```

### View Logs

```bash
# View logs of primary instance
kubectl logs -n pg my-pgsql-cluster-1 -f

# View logs of all instances
kubectl logs -n pg -l cnpg.io/cluster=my-pgsql-cluster --all-containers=true
```

## Backup and Recovery

CloudNativePG supports automated backups. To configure backups, you need to add backup configuration to the cluster spec:

```yaml
cluster:
  backup:
    barmanObjectStore:
      destinationPath: s3://my-bucket/backups
      s3Credentials:
        accessKeyId:
          name: aws-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-credentials
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
    retentionPolicy: "30d"
```

### Manual Backup

```bash
kubectl create -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: manual-backup-$(date +%Y%m%d-%H%M%S)
  namespace: pg
spec:
  cluster:
    name: my-pgsql-cluster
EOF
```

## Scaling

### Scale Up Replicas

```bash
helm upgrade my-postgres ./postgres-cnpg \
  --set cluster.instances=3 \
  --reuse-values
```

Or directly:
```bash
kubectl patch cluster my-pgsql-cluster -n pg \
  --type='json' -p='[{"op": "replace", "path": "/spec/instances", "value": 3}]'
```

### Scale Down

```bash
helm upgrade my-postgres ./postgres-cnpg \
  --set cluster.instances=1 \
  --reuse-values
```

> **Note:** CloudNativePG automatically handles the failover process during scaling operations.

## Monitoring

### Prometheus Integration

CloudNativePG exposes Prometheus metrics by default. To scrape metrics:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-pgsql-cluster-metrics
  namespace: pg
  labels:
    prometheus: "true"
spec:
  type: ClusterIP
  ports:
  - name: metrics
    port: 9187
    targetPort: 9187
  selector:
    cnpg.io/cluster: my-pgsql-cluster
```

### Key Metrics

- `cnpg_pg_replication_lag` - Replication lag in bytes
- `cnpg_pg_database_size_bytes` - Database size
- `cnpg_pg_stat_archiver` - Archive status
- `cnpg_backends_total` - Number of backends

## Upgrading

### Upgrade PostgreSQL Version

```bash
helm upgrade my-postgres ./postgres-cnpg \
  --set cluster.image=ghcr.io/cloudnative-pg/postgresql:15.2 \
  --reuse-values
```

CloudNativePG performs rolling updates automatically based on the `primaryUpdateStrategy`.

### Upgrade Chart Version

```bash
helm upgrade my-postgres ./postgres-cnpg -f values.yaml
```

### Rollback

```bash
helm rollback my-postgres
```

## High Availability Features

### Automatic Failover

CloudNativePG automatically promotes a replica to primary if the primary fails:

- **Failover time:** Typically 30-60 seconds
- **Zero data loss:** When using synchronous replication
- **Automatic:** No manual intervention required

### Switchover (Planned Maintenance)

```bash
kubectl cnpg promote my-pgsql-cluster-2 -n pg
```

### Connection Pooling

Consider using PgBouncer for connection pooling:

```yaml
cluster:
  connectionPooler:
    enabled: true
    instances: 2
    type: pgbouncer
    pgbouncer:
      poolMode: transaction
      parameters:
        max_client_conn: "1000"
        default_pool_size: "25"
```

## Troubleshooting

### Cluster Not Starting

Check operator logs:
```bash
kubectl logs -n cnpg-system deployment/cnpg-controller-manager
```

Check cluster events:
```bash
kubectl describe cluster my-pgsql-cluster -n pg
```

### Connection Issues

Verify services:
```bash
kubectl get svc -n pg
kubectl get endpoints -n pg
```

Check pg_hba configuration:
```bash
kubectl exec -n pg my-pgsql-cluster-1 -- psql -U postgres -c "SELECT * FROM pg_hba_file_rules;"
```

### Storage Issues

Check PVCs:
```bash
kubectl get pvc -n pg
```

Check storage class:
```bash
kubectl get storageclass longhorn
```

### Replication Lag

Check replication status:
```bash
kubectl exec -n pg my-pgsql-cluster-1 -- \
  psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

### Pod Crashes

View pod logs:
```bash
kubectl logs -n pg my-pgsql-cluster-1 --previous
```

Check resource limits:
```bash
kubectl describe pod -n pg my-pgsql-cluster-1
```

## Production Recommendations

### Resource Allocation

```yaml
cluster:
  postgresql:
    parameters:
      shared_buffers: "2GB"          # 25% of total RAM
      effective_cache_size: "6GB"     # 75% of total RAM
      maintenance_work_mem: "512MB"   # RAM / 16
      work_mem: "20MB"                # RAM / (max_connections * 2)
      max_wal_size: "4GB"
```

### Security Best Practices

1. **Change Default Passwords:**
   ```bash
   kubectl create secret generic pg-superuser \
     -n pg \
     --from-literal=username=postgres \
     --from-literal=password=$(openssl rand -base64 32)
   ```

2. **Restrict Network Access:**
   ```yaml
   cluster:
     postgresql:
       pg_hba:
         - hostssl all all 10.0.0.0/8 scram-sha-256
         - hostnossl all all 0.0.0.0/0 reject
   ```

3. **Enable TLS/SSL:**
   ```yaml
   cluster:
     certificates:
       serverTLSSecret: my-postgres-tls
       serverCASecret: my-postgres-ca
   ```

### Storage Recommendations

- **Development:** 5-10Gi
- **Production:** 50Gi+ with SSD-backed storage
- **Use StorageClass with good IOPS:** NVMe, SSD, or high-performance cloud storage
- **Enable volume expansion:** Ensure StorageClass supports expansion

### High Availability Setup

```yaml
cluster:
  instances: 3                          # Minimum for HA
  minSyncReplicas: 1                    # Synchronous replication
  maxSyncReplicas: 1
  postgresql:
    parameters:
      synchronous_commit: "on"
      max_wal_senders: "10"
      max_replication_slots: "10"
```

### Monitoring Setup

1. Enable Prometheus ServiceMonitor
2. Set up Grafana dashboard (CloudNativePG provides official dashboards)
3. Configure alerting for:
   - High replication lag
   - Low disk space
   - Connection pool exhaustion
   - Failed backups

### Backup Strategy

```yaml
cluster:
  backup:
    retentionPolicy: "30d"
    barmanObjectStore:
      wal:
        compression: gzip
        maxParallel: 8
      data:
        compression: gzip
```

Schedule regular backups:
```bash
# Daily backup at 2 AM
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: daily-backup
  namespace: pg
spec:
  schedule: "0 2 * * *"
  cluster:
    name: my-pgsql-cluster
```

## Uninstalling

### Remove the Cluster

```bash
helm uninstall my-postgres -n pg
```

### Clean Up PVCs (Optional)

```bash
kubectl delete pvc -n pg -l cnpg.io/cluster=my-pgsql-cluster
```

### Remove Secrets

```bash
kubectl delete secret pg-superuser pg-app-user -n pg
```

> **Warning:** Deleting PVCs will permanently delete all data!

## Architecture

### Components

```
┌─────────────────────────────────────────┐
│     CloudNativePG Operator              │
│  (Manages PostgreSQL Cluster Lifecycle) │
└─────────────────────────────────────────┘
                    │
                    ├─ Creates & Manages
                    ↓
┌─────────────────────────────────────────┐
│        PostgreSQL Cluster               │
│                                         │
│  ┌──────────┐  ┌──────────┐             │
│  │ Primary  │  │ Replica  │             │
│  │  Pod-1   │→ │  Pod-2   │             │
│  └──────────┘  └──────────┘             │
│       │             │                   │
│       ↓             ↓                   │
│     PVC           PVC                   │
└─────────────────────────────────────────┘
                    │
                    ├─ Services
                    ↓
┌─────────────────────────────────────────┐
│  • -rw (read-write to primary)          │
│  • -ro (read-only to replicas)          │
│  • -r  (read to any instance)           │
└─────────────────────────────────────────┘
```

### Network Services

- **`<cluster-name>-rw`**: Routes to primary (read-write operations)
- **`<cluster-name>-ro`**: Routes to replicas (read-only operations)
- **`<cluster-name>-r`**: Routes to any instance (read operations)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Resources

- [CloudNativePG Documentation](https://cloudnative-pg.io/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [CloudNativePG GitHub](https://github.com/cloudnative-pg/cloudnative-pg)

## License

This Helm chart is open source and available under the Apache 2.0 license.

## Support

For issues and questions:
- Create an issue in the GitHub repository
- Check CloudNativePG documentation
- Join CloudNativePG Slack community

## Version History

- **0.1.0** - Initial release with PostgreSQL 15.1 support

## Authors

PHAM THANH TUNG

---

**Note:** This chart requires the CloudNativePG operator to be installed separately. The operator manages the PostgreSQL cluster lifecycle, automated failover, and backup/recovery operations.
