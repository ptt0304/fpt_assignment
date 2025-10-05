# Kubernetes Helm Charts Collection

A collection of testing-only Helm charts for deploying popular databases and application stacks on Kubernetes.

## 📦 Available Charts

| Chart | Description | Version | App Version |
|-------|-------------|---------|-------------|
| [Cassandra](#cassandra) | Apache Cassandra NoSQL database | 0.1.0 | 4.1 |
| [PostgreSQL (CNPG)](#postgresql-cloudnativepg) | PostgreSQL with CloudNativePG operator | 0.1.0 | 15.1 |
| [LAMP Stack](#lamp-stack) | Linux, Apache, MySQL, PHP stack | 0.1.0 | PHP 8.2, MySQL 8.0 |

## 🚀 Quick Start

### Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- kubectl configured to access your cluster

### Installation

```bash
# Clone the repository
git clone <your-repo-url>
cd <repo-name>

# Install a chart
helm install my-release ./<chart-name>
```

## 📊 Charts Overview

### Cassandra

Apache Cassandra is a highly scalable, distributed NoSQL database designed for handling large amounts of data across many servers.

**Key Features:**
- StatefulSet deployment with stable network identities
- Persistent storage support (Longhorn)
- Configurable cluster settings (tokens, replication)
- Multi-replica deployment with automatic seed discovery
- Headless service for internal communication

**Quick Install:**
```bash
cd cassandra
helm install cassandra-cluster .
```

**Access:**
```bash
# CQL Shell
kubectl exec -it statefulset/cassandra-cluster-cassandra -- cqlsh

# Port Forward
kubectl port-forward svc/cassandra-cluster-cassandra 9042:9042
```

**Use Cases:**
- Time-series data
- IoT sensor data
- High-write workloads
- Distributed systems requiring eventual consistency

📖 [Full Documentation](./cassandra/README.md)

---

### PostgreSQL (CloudNativePG)

PostgreSQL deployed with CloudNativePG operator for automated management, high availability, and backup/recovery.

**Key Features:**
- Automatic failover and self-healing
- Built-in backup and recovery (Barman)
- Read-write and read-only services
- Optimized PostgreSQL parameters
- TLS/SSL support
- Connection pooling (PgBouncer)

**Prerequisites:**
```bash
# Install CloudNativePG operator first
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.0.yaml
```

**Quick Install:**
```bash
cd postgres-cnpg
kubectl create namespace pg
helm install my-postgres .
```

**Access:**
```bash
# Connection string
postgresql://app_user:app-user-password@my-pgsql-cluster-rw.pg.svc.cluster.local:5432/my_app_db

# Port Forward
kubectl port-forward -n pg svc/my-pgsql-cluster-rw 5432:5432
```

**Use Cases:**
- ACID-compliant transactions
- Complex queries and joins
- Mission-critical applications
- Data warehousing
- GIS applications (PostGIS)

📖 [Full Documentation](./postgres-cnpg/README.md)

---

### LAMP Stack

Traditional LAMP (Linux, Apache, MySQL, PHP) stack for running PHP applications on Kubernetes.

**Key Features:**
- PHP 8.2 with Apache web server
- MySQL 8.0 database
- NodePort for external access
- Simple configuration
- Perfect for WordPress, Laravel, and PHP apps

**Quick Install:**
```bash
cd lamp
helm install lamp-stack .
```

**Access:**
```bash
# Apache (NodePort)
http://<node-ip>:32100

# MySQL connection from PHP
mysql://user:userpassword@mysql:3306/mydb
```

**Use Cases:**
- WordPress websites
- PHP web applications
- Content management systems
- E-commerce platforms
- Legacy PHP applications

📖 [Full Documentation](./lamp/README.md)

---

## 🔧 Configuration

Each chart has its own `values.yaml` file for customization. Common parameters:

### Storage Configuration

```yaml
# Cassandra
storage:
  enabled: true
  size: 10Gi
  className: longhorn

# PostgreSQL
cluster:
  storage:
    size: 20Gi
    storageClass: longhorn

# MySQL (LAMP)
# Add PVC configuration (see chart documentation)
```

### Replica Configuration

```yaml
# Cassandra
replicaCount: 3

# PostgreSQL
cluster:
  instances: 3

# Apache (LAMP)
# Update deployment replicas
```

### Security Configuration

```yaml
# Cassandra
secret:
  username: cassandra
  password: <your-secure-password>

# PostgreSQL
# Update secrets.yaml with secure passwords

# MySQL (LAMP)
mysql:
  rootPassword: <your-secure-password>
  password: <your-secure-password>
```

## 📋 Comparison Matrix

| Feature | Cassandra | PostgreSQL | MySQL (LAMP) |
|---------|-----------|------------|--------------|
| **Type** | NoSQL (Wide Column) | SQL (Relational) | SQL (Relational) |
| **HA Built-in** | ✅ Yes | ✅ Yes (with CNPG) | ❌ No (basic chart) |
| **Auto Failover** | ✅ Yes | ✅ Yes | ❌ No |
| **Backup/Recovery** | Manual | ✅ Automated | Manual |
| **Complexity** | High | Medium | Low |
| **Best For** | Big Data, IoT | ACID, Complex Queries | PHP Apps, WordPress |
| **Scaling** | Horizontal | Vertical + Read Replicas | Limited |
| **Consistency** | Eventual | Strong | Strong |

## 🛠️ Common Operations

### Scaling

```bash
# Cassandra
helm upgrade cassandra-cluster ./cassandra --set replicaCount=5

# PostgreSQL
helm upgrade my-postgres ./postgres-cnpg --set cluster.instances=3

# LAMP (Apache only)
kubectl scale deployment apache --replicas=3
```

### Upgrading

```bash
# Upgrade chart
helm upgrade <release-name> ./<chart-name> -f values.yaml

# Rollback if needed
helm rollback <release-name>
```

### Backup

```bash
# Cassandra
kubectl exec -it statefulset/cassandra-cluster-cassandra -- nodetool snapshot

# PostgreSQL (automatic with CNPG)
kubectl create -f backup.yaml

# MySQL
kubectl exec -it <mysql-pod> -- mysqldump -uroot -p<password> mydb > backup.sql
```

### Monitoring

```bash
# Check pod status
kubectl get pods

# View logs
kubectl logs -f <pod-name>

# Check cluster health
# Cassandra
kubectl exec -it statefulset/cassandra-cluster-cassandra -- nodetool status

# PostgreSQL
kubectl get cluster -n pg
```

## 📚 Documentation Structure

Each chart includes comprehensive documentation:

```
chart-name/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration
├── README.md              # Detailed documentation
└── templates/             # Kubernetes manifests
    ├── deployment.yaml
    ├── service.yaml
    ├── statefulset.yaml
    └── ...
```

## 🔒 Security Best Practices

### For All Charts:

1. **Change Default Passwords**
   ```bash
   # Generate secure passwords
   openssl rand -base64 32
   ```

2. **Use Kubernetes Secrets**
   ```bash
   kubectl create secret generic db-credentials \
     --from-literal=password=$(openssl rand -base64 32)
   ```

3. **Enable Network Policies**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: database-policy
   spec:
     podSelector:
       matchLabels:
         app: database
     ingress:
     - from:
       - podSelector:
           matchLabels:
             app: application
   ```

4. **Use TLS/SSL**
   - Enable encryption in transit
   - Use cert-manager for certificates

5. **Set Resource Limits**
   ```yaml
   resources:
     requests:
       memory: "2Gi"
       cpu: "1000m"
     limits:
       memory: "4Gi"
       cpu: "2000m"
   ```

6. **Enable Pod Security Policies**

7. **Regular Updates**
   - Keep images updated
   - Apply security patches

8. **Implement RBAC**
   - Least privilege access
   - Service accounts with limited permissions

## 🐛 Troubleshooting

### Common Issues

**Pods Not Starting:**
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

**Storage Issues:**
```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
```

**Network Issues:**
```bash
kubectl get svc
kubectl get endpoints
```

**Permission Issues:**
```bash
kubectl get psp
kubectl describe psp <psp-name>
```

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

### Development Guidelines

- Follow Helm best practices
- Update documentation
- Add examples
- Test on different Kubernetes versions
- Include upgrade/rollback tests

## 📄 License

This collection is open source and available under the Apache 2.0 license.

## 🆘 Support

- 📖 Check individual chart documentation
- 🐛 Report issues on GitHub
- 💬 Join community discussions
- 📧 Contact maintainers

## 🗺️ Roadmap

- [ ] Add Redis chart
- [ ] Add MongoDB chart
- [ ] Add Elasticsearch chart
- [ ] Add monitoring stack (Prometheus/Grafana)
- [ ] Add CI/CD examples
- [ ] Add Terraform modules
- [ ] Improve backup/restore automation
- [ ] Add disaster recovery procedures

## ⭐ Star History

If you find these charts useful, please consider giving the repository a star!

## 👥 Authors

PHAM THANH TUNG

---

**Quick Links:**
- [Cassandra Documentation](./cassandra/README.md)
- [PostgreSQL Documentation](./postgres-cnpg/README.md)
- [LAMP Stack Documentation](./lamp/README.md)

**Last Updated:** October 2025
