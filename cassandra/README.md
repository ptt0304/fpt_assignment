# Apache Cassandra Helm Chart

A production-ready Helm chart for deploying Apache Cassandra on Kubernetes with StatefulSet support.

## Overview

This Helm chart deploys a highly available Apache Cassandra cluster on Kubernetes. It provides persistent storage, configurable cluster settings, and easy scalability for your distributed database needs.

## Features

- ✅ StatefulSet-based deployment for stable network identities
- ✅ Persistent storage support with configurable storage classes
- ✅ Headless service for internal cluster communication
- ✅ Configurable cluster settings (cluster name, tokens)
- ✅ Secret management for authentication
- ✅ Resource management and limits
- ✅ Optional Ingress support
- ✅ Multi-replica deployment with automatic seed discovery

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PersistentVolume provisioner support (if persistent storage is enabled)
- StorageClass configured (default: Longhorn)

## Installation

### Add Helm Repository

```bash
helm repo add <your-repo-name> <your-repo-url>
helm repo update
```

### Install the Chart

```bash
helm install my-cassandra ./cassandra
```

Or with custom values:

```bash
helm install my-cassandra ./cassandra -f custom-values.yaml
```

### Install in a Specific Namespace

```bash
kubectl create namespace cassandra
helm install my-cassandra ./cassandra -n cassandra
```

## Configuration

### Global Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of Cassandra replicas | `3` |

### Image Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Cassandra image repository | `cassandra` |
| `image.tag` | Cassandra image tag | `4.1` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |

### Service Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | CQL service port | `9042` |

### Storage Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `storage.enabled` | Enable persistent storage | `true` |
| `storage.size` | Size of persistent volume | `1Gi` |
| `storage.className` | StorageClass name | `longhorn` |

### Cassandra Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `config.clusterName` | Name of the Cassandra cluster | `CassandraCluster` |
| `config.numTokens` | Number of tokens per node | `256` |

### Authentication

| Parameter | Description | Default |
|-----------|-------------|---------|
| `secret.username` | Cassandra username | `cassandra` |
| `secret.password` | Cassandra password | `cassandra` |

> **Warning:** Change default credentials before deploying to production!

### Ingress Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `ingress.enabled` | Enable ingress | `false` |
| `ingress.className` | Ingress class name | `""` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | Ingress hosts configuration | See values.yaml |
| `ingress.tls` | Ingress TLS configuration | `[]` |

### Resource Limits

| Parameter | Description | Default |
|-----------|-------------|---------|
| `resources` | CPU/Memory resource requests/limits | `{}` |

## Usage Examples

### Basic Installation

```bash
helm install cassandra-cluster ./cassandra
```

### Custom Storage Size

```bash
helm install cassandra-cluster ./cassandra \
  --set storage.size=10Gi
```

### High Availability Cluster

```bash
helm install cassandra-cluster ./cassandra \
  --set replicaCount=5 \
  --set storage.size=20Gi \
  --set resources.requests.memory=4Gi \
  --set resources.requests.cpu=2000m
```

### With Custom Credentials

```bash
helm install cassandra-cluster ./cassandra \
  --set secret.username=admin \
  --set secret.password=strongpassword123
```

## Accessing Cassandra

### Using CQL Shell

Connect to a pod directly:

```bash
kubectl exec -it statefulset/cassandra-cluster-cassandra -n <namespace> -- cqlsh
```

### Port Forwarding

Forward the CQL port to your local machine:

```bash
kubectl port-forward svc/cassandra-cluster-cassandra -n <namespace> 9042:9042
```

Then connect with your favorite Cassandra client at `127.0.0.1:9042`

### Check Cluster Status

```bash
kubectl exec -it statefulset/cassandra-cluster-cassandra -n <namespace> -- nodetool status
```

## Architecture

This chart deploys Cassandra as a StatefulSet with the following components:

- **StatefulSet**: Manages the Cassandra pods with stable network identities
- **Headless Service**: Enables pod-to-pod communication within the cluster
- **ConfigMap**: Stores cluster configuration (cluster name, tokens)
- **Secret**: Stores authentication credentials (base64 encoded)
- **PersistentVolumeClaims**: Provides persistent storage for each Cassandra node

### Network Ports

| Port | Name | Description |
|------|------|-------------|
| 7000 | intra-node | Inter-node cluster communication |
| 7001 | tls-intra-node | TLS inter-node communication |
| 7199 | jmx | JMX monitoring |
| 9042 | cql | CQL native transport |

## Scaling

### Scale Up

```bash
helm upgrade cassandra-cluster ./cassandra --set replicaCount=5
```

### Scale Down

```bash
helm upgrade cassandra-cluster ./cassandra --set replicaCount=2
```

> **Note:** When scaling down, ensure proper decommissioning of nodes to prevent data loss.

## Backup and Recovery

Cassandra data is stored in PersistentVolumes mounted at `/var/lib/cassandra` on each pod. Ensure your StorageClass supports snapshots for backup purposes.

## Upgrading

### Upgrade the Chart

```bash
helm upgrade cassandra-cluster ./cassandra -f values.yaml
```

### Rollback

```bash
helm rollback cassandra-cluster
```

## Uninstalling

```bash
helm uninstall cassandra-cluster -n <namespace>
```

> **Warning:** This will delete all Cassandra pods. PersistentVolumeClaims may be retained depending on your configuration.

## Troubleshooting

### Pods Not Starting

Check pod logs:
```bash
kubectl logs -f statefulset/cassandra-cluster-cassandra -n <namespace>
```

### Storage Issues

Verify PVCs are bound:
```bash
kubectl get pvc -n <namespace>
```

### Network Issues

Check service endpoints:
```bash
kubectl get endpoints cassandra-cluster-cassandra -n <namespace>
```

### Seed Node Issues

Ensure the first pod (`-0`) is running and healthy before other pods join the cluster.

## Production Recommendations

1. **Storage**: Use at least 10Gi per node, preferably SSD-backed storage
2. **Resources**: Allocate minimum 4GB RAM and 2 CPU cores per pod
3. **Replicas**: Run at least 3 replicas for production workloads
4. **Credentials**: Always change default username/password
5. **Monitoring**: Integrate with Prometheus/Grafana for metrics
6. **Backups**: Implement regular backup strategy using nodetool snapshot
7. **Network Policies**: Restrict access to Cassandra ports

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This Helm chart is open source and available under the Apache 2.0 license.

## Support

For issues and questions:
- Create an issue in the GitHub repository
- Check Cassandra official documentation: https://cassandra.apache.org/doc/

## Version History

- **0.1.0** - Initial release with basic Cassandra deployment support

## Authors

Pham Thanh Tung
---

**Note:** This chart uses the official Apache Cassandra Docker image from Docker Hub.
