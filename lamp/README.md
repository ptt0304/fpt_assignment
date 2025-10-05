# LAMP Stack Helm Chart

A simple yet powerful Helm chart for deploying a complete LAMP (Linux, Apache, MySQL, PHP) stack on Kubernetes.

## Overview

This Helm chart provides a quick and easy way to deploy a traditional LAMP stack in Kubernetes. It includes Apache with PHP 8.2 and MySQL 8.0, making it perfect for running PHP applications, WordPress sites, or any PHP-based web applications.

## Features

- ✅ **Apache with PHP 8.2** - Modern PHP version with Apache web server
- ✅ **MySQL 8.0** - Latest stable MySQL database
- ✅ **Simple Configuration** - Easy-to-use values for quick deployment
- ✅ **NodePort Access** - Direct external access to Apache (configurable)
- ✅ **Environment Variables** - MySQL configuration through environment variables
- ✅ **Kubernetes Native** - Uses standard Deployments and Services

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- kubectl configured to access your cluster

## Quick Start

### Install the Chart

```bash
helm install my-lamp ./lamp
```

### Access Your Application

If using NodePort (default), access Apache at:
```
http://<node-ip>:32100
```

Find your node IP:
```bash
kubectl get nodes -o wide
```

## Architecture

```
┌─────────────────────────────────────────┐
│           External Access               │
│        http://<node-ip>:32100           │
└─────────────────┬───────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────┐
│      Apache Service (NodePort)          │
│            Port: 80                     │
└─────────────────┬───────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────┐
│      Apache + PHP 8.2 Deployment        │
│         Container Port: 80              │
└─────────────────┬───────────────────────┘
                  │
                  │ Connects to
                  ↓
┌─────────────────────────────────────────┐
│      MySQL Service (ClusterIP)          │
│            Port: 3306                   │
└─────────────────┬───────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────┐
│        MySQL 8.0 Deployment             │
│         Container Port: 3306            │
│                                         │
│  Database: mydb                         │
│  User: user / userpassword              │
│  Root: root / rootpassword              │
└─────────────────────────────────────────┘
```

## Configuration

### MySQL Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `mysql.image` | MySQL container image | `mysql:8.0` |
| `mysql.rootPassword` | MySQL root password | `rootpassword` |
| `mysql.database` | Initial database name | `mydb` |
| `mysql.user` | MySQL user | `user` |
| `mysql.password` | MySQL user password | `userpassword` |
| `mysql.service.port` | MySQL service port | `3306` |

### Apache/PHP Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `apache.image` | PHP with Apache image | `php:8.2-apache` |
| `apache.service.type` | Kubernetes service type | `NodePort` |
| `apache.service.port` | Apache service port | `80` |
| `apache.service.nodePort` | NodePort for external access | `32100` |

## Usage Examples

### Basic Installation

```bash
helm install lamp-stack ./lamp
```

### Custom MySQL Credentials

```bash
helm install lamp-stack ./lamp \
  --set mysql.rootPassword=MySecureRootPass123 \
  --set mysql.password=MySecureUserPass123 \
  --set mysql.database=myapp_db \
  --set mysql.user=myapp_user
```

### Using LoadBalancer Instead of NodePort

```bash
helm install lamp-stack ./lamp \
  --set apache.service.type=LoadBalancer
```

### Using ClusterIP with Ingress

```bash
helm install lamp-stack ./lamp \
  --set apache.service.type=ClusterIP
```

Then create an Ingress resource separately.

### Custom PHP Version

```bash
helm install lamp-stack ./lamp \
  --set apache.image=php:8.1-apache
```

### Custom MySQL Version

```bash
helm install lamp-stack ./lamp \
  --set mysql.image=mysql:5.7
```

## Connecting to MySQL

### From Apache/PHP Container

Use the following connection parameters in your PHP application:

```php
<?php
$servername = "mysql";  // Service name
$username = "user";     // From values.yaml
$password = "userpassword";  // From values.yaml
$database = "mydb";     // From values.yaml

$conn = new mysqli($servername, $username, $password, $database);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}
echo "Connected successfully";
?>
```

### Connection String

```
mysql://user:userpassword@mysql:3306/mydb
```

### From kubectl

```bash
# Get MySQL pod name
kubectl get pods -l app=mysql

# Connect to MySQL
kubectl exec -it <mysql-pod-name> -- mysql -uroot -prootpassword

# Or connect as regular user
kubectl exec -it <mysql-pod-name> -- mysql -uuser -puserpassword mydb
```

### Port Forward for Local Access

```bash
kubectl port-forward svc/mysql 3306:3306
```

Then connect from your local machine:
```bash
mysql -h 127.0.0.1 -u user -puserpassword mydb
```

## Deploying Your PHP Application

### Method 1: Using Custom Docker Image

Build your own image with your PHP code:

```dockerfile
FROM php:8.2-apache

# Install MySQL extension
RUN docker-php-ext-install mysqli pdo pdo_mysql

# Copy your application
COPY ./src /var/www/html/

# Set permissions
RUN chown -R www-data:www-data /var/www/html
```

Then update the chart:
```bash
helm upgrade lamp-stack ./lamp \
  --set apache.image=your-registry/your-php-app:latest
```

### Method 2: Using ConfigMap for Simple Apps

Create a ConfigMap with your PHP files:

```bash
kubectl create configmap php-code \
  --from-file=index.php=./index.php \
  --from-file=config.php=./config.php
```

Update `apache-deployment.yaml` to mount the ConfigMap:
```yaml
volumeMounts:
  - name: php-code
    mountPath: /var/www/html
volumes:
  - name: php-code
    configMap:
      name: php-code
```

### Method 3: Using PersistentVolume

For larger applications, use PersistentVolume to store your code.

## Installing WordPress

### Step 1: Deploy LAMP Stack

```bash
helm install wordpress ./lamp \
  --set mysql.database=wordpress \
  --set mysql.user=wpuser \
  --set mysql.password=wppassword \
  --set mysql.rootPassword=wprootpass
```

### Step 2: Create WordPress Dockerfile

```dockerfile
FROM php:8.2-apache

# Install dependencies
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libzip-dev \
    && docker-php-ext-configure gd --with-jpeg \
    && docker-php-ext-install gd mysqli zip pdo pdo_mysql

# Download WordPress
RUN curl -O https://wordpress.org/latest.tar.gz \
    && tar -xzf latest.tar.gz \
    && mv wordpress/* /var/www/html/ \
    && rm -rf wordpress latest.tar.gz \
    && chown -R www-data:www-data /var/www/html

# Enable Apache rewrite module
RUN a2enmod rewrite
```

### Step 3: Update Deployment

Build and push your image, then:
```bash
helm upgrade wordpress ./lamp \
  --set apache.image=your-registry/wordpress:latest \
  --reuse-values
```

## Common PHP Extensions

The default `php:8.2-apache` image is minimal. You may need to install additional extensions:

```dockerfile
FROM php:8.2-apache

# Common extensions
RUN docker-php-ext-install \
    mysqli \
    pdo \
    pdo_mysql \
    gd \
    zip \
    mbstring \
    bcmath

# Enable Apache modules
RUN a2enmod rewrite
```

Popular extensions:
- `mysqli` - MySQL Improved Extension
- `pdo_mysql` - PDO MySQL Driver
- `gd` - Image Processing
- `zip` - ZIP Archive
- `mbstring` - Multibyte String
- `xml` - XML Parser
- `curl` - cURL
- `json` - JSON
- `bcmath` - BC Math

## Persistent Storage for MySQL

⚠️ **Important:** The current configuration does not use persistent storage for MySQL. Data will be lost if the pod restarts!

### Adding PersistentVolume

Update `mysql-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: {{ .Values.mysql.image }}
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: {{ .Values.mysql.rootPassword | quote }}
        - name: MYSQL_DATABASE
          value: {{ .Values.mysql.database | quote }}
        - name: MYSQL_USER
          value: {{ .Values.mysql.user | quote }}
        - name: MYSQL_PASSWORD
          value: {{ .Values.mysql.password | quote }}
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
```

Create PVC:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: longhorn  # Or your StorageClass
```

## Monitoring and Logs

### View Apache Logs

```bash
# Get pod name
kubectl get pods -l app=apache

# View logs
kubectl logs -f <apache-pod-name>
```

### View MySQL Logs

```bash
# Get pod name
kubectl get pods -l app=mysql

# View logs
kubectl logs -f <mysql-pod-name>
```

### Execute Commands in Containers

```bash
# Apache container
kubectl exec -it <apache-pod-name> -- bash

# MySQL container
kubectl exec -it <mysql-pod-name> -- bash
```

## Troubleshooting

### Apache Cannot Connect to MySQL

**Symptom:** PHP shows "Connection refused" error

**Solution:** 
1. Check if MySQL service is running:
   ```bash
   kubectl get svc mysql
   kubectl get pods -l app=mysql
   ```

2. Verify MySQL is ready:
   ```bash
   kubectl logs <mysql-pod-name>
   ```

3. Test connection from Apache pod:
   ```bash
   kubectl exec -it <apache-pod-name> -- ping mysql
   ```

### Cannot Access Apache Externally

**Symptom:** Cannot reach http://node-ip:32100

**Solutions:**

1. Check if NodePort is open on firewall:
   ```bash
   # On node
   sudo firewall-cmd --add-port=32100/tcp --permanent
   sudo firewall-cmd --reload
   ```

2. Verify service:
   ```bash
   kubectl get svc apache
   kubectl describe svc apache
   ```

3. Check pod status:
   ```bash
   kubectl get pods -l app=apache
   kubectl logs <apache-pod-name>
   ```

### MySQL Pod Crashes

**Common causes:**

1. **Insufficient memory** - MySQL needs at least 512MB RAM
   
2. **Permission issues** - Check pod events:
   ```bash
   kubectl describe pod <mysql-pod-name>
   ```

3. **Corrupt data** - If using PersistentVolume, data may be corrupted:
   ```bash
   kubectl delete pod <mysql-pod-name>  # Restart pod
   ```

### PHP Extensions Missing

**Symptom:** PHP errors about missing functions (mysqli_connect, etc.)

**Solution:** Build custom image with required extensions (see "Common PHP Extensions" section)

## Security Considerations

### ⚠️ Production Warnings

This chart is designed for development and testing. For production use:

1. **Change Default Passwords:**
   ```bash
   helm install lamp-stack ./lamp \
     --set mysql.rootPassword=$(openssl rand -base64 32) \
     --set mysql.password=$(openssl rand -base64 32)
   ```

2. **Use Secrets Instead of Values:**
   Create secrets manually:
   ```bash
   kubectl create secret generic mysql-root \
     --from-literal=password=$(openssl rand -base64 32)
   
   kubectl create secret generic mysql-user \
     --from-literal=username=myuser \
     --from-literal=password=$(openssl rand -base64 32)
   ```

3. **Enable TLS/SSL:**
   - Use cert-manager for certificates
   - Configure Apache for HTTPS
   - Update service to use port 443

4. **Use Ingress Instead of NodePort:**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: lamp-ingress
   spec:
     rules:
     - host: myapp.example.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: apache
               port:
                 number: 80
   ```

5. **Add Resource Limits:**
   ```yaml
   resources:
     requests:
       memory: "512Mi"
       cpu: "250m"
     limits:
       memory: "1Gi"
       cpu: "500m"
   ```

6. **Enable Network Policies:**
   Restrict MySQL access to only Apache pods

7. **Use Persistent Storage:**
   Always use PersistentVolumes for MySQL data

8. **Regular Backups:**
   Implement automated MySQL backups

## Scaling

### Scale Apache

```bash
# Update apache-deployment.yaml to support scaling
kubectl scale deployment apache --replicas=3
```

### MySQL High Availability

For production, consider:
- MySQL with replication (Master-Slave)
- Using CloudNativePG for PostgreSQL instead
- Managed database services (RDS, Cloud SQL)

## Performance Tuning

### MySQL Optimization

Add to `mysql-deployment.yaml` env:
```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    value: {{ .Values.mysql.rootPassword | quote }}
  - name: MYSQL_DATABASE
    value: {{ .Values.mysql.database | quote }}
  - name: MYSQL_USER
    value: {{ .Values.mysql.user | quote }}
  - name: MYSQL_PASSWORD
    value: {{ .Values.mysql.password | quote }}
  # Performance tuning
  - name: MYSQL_INNODB_BUFFER_POOL_SIZE
    value: "512M"
  - name: MYSQL_MAX_CONNECTIONS
    value: "200"
```

### Apache/PHP Tuning

Custom Apache configuration via ConfigMap:
```bash
kubectl create configmap apache-config \
  --from-file=apache2.conf
```

## Upgrading

### Upgrade the Chart

```bash
helm upgrade lamp-stack ./lamp -f values.yaml
```

### Upgrade MySQL Version

```bash
helm upgrade lamp-stack ./lamp \
  --set mysql.image=mysql:8.1 \
  --reuse-values
```

⚠️ **Warning:** Always backup MySQL data before upgrading!

### Rollback

```bash
helm rollback lamp-stack
```

## Uninstalling

### Remove the Chart

```bash
helm uninstall lamp-stack
```

### Clean Up Resources

```bash
# Delete PVCs if created
kubectl delete pvc mysql-pvc

# Delete ConfigMaps if created
kubectl delete configmap php-code apache-config
```

## Examples and Templates

### Example PHP Info Page

Create `index.php`:
```php
<?php
phpinfo();
?>
```

### Example MySQL Connection Test

Create `test-db.php`:
```php
<?php
$servername = "mysql";
$username = "user";
$password = "userpassword";
$database = "mydb";

$conn = new mysqli($servername, $username, $password, $database);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

echo "Connected successfully to MySQL!<br>";
echo "Server version: " . $conn->server_info . "<br>";
echo "Database: " . $database . "<br>";

$conn->close();
?>
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Resources

- [PHP Official Documentation](https://www.php.net/docs.php)
- [MySQL Documentation](https://dev.mysql.com/doc/)
- [Apache HTTP Server](https://httpd.apache.org/docs/)
- [PHP Docker Images](https://hub.docker.com/_/php)
- [MySQL Docker Images](https://hub.docker.com/_/mysql)

## License

This Helm chart is open source and available under the Apache 2.0 license.

## Support

For issues and questions:
- Create an issue in the GitHub repository
- Check the troubleshooting section above
- Review PHP, MySQL, and Apache documentation

## Version History

- **0.1.0** - Initial release with PHP 8.2 and MySQL 8.0

## Authors

PHAM THANH TUNG

---

**Note:** This is a basic LAMP stack suitable for development and testing. For production deployments, implement security hardening, persistent storage, backups, and monitoring solutions.
