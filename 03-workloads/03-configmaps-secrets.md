# ConfigMaps and Secrets

Kubernetes provides two objects for storing application configuration: **ConfigMaps** for non-sensitive data and **Secrets** for sensitive data. Both allow you to decouple configuration from container images.

---

## ConfigMaps

### Creating ConfigMaps

#### From Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100 \
  --from-literal=DB_HOST=postgres.default.svc.cluster.local
```

#### From a File

```bash
# config.properties file
kubectl create configmap app-config --from-file=config.properties

# Custom key name
kubectl create configmap app-config --from-file=myconfig=config.properties
```

#### From an Environment File

```bash
# .env file format: KEY=VALUE
kubectl create configmap app-config --from-env-file=app.env
```

#### Declarative YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  DB_HOST: "postgres.default.svc.cluster.local"

  # Multi-line configuration file content
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      pool_size: 10
      connect_timeout: 5s

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://localhost:8080;
      }
    }
```

### Using ConfigMaps in Pods

#### As Environment Variables (Individual Keys)

```yaml
spec:
  containers:
  - name: app
    image: my-app:1.0
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
```

#### Inject All Keys as Environment Variables

```yaml
spec:
  containers:
  - name: app
    image: my-app:1.0
    envFrom:
    - configMapRef:
        name: app-config
```

> 💡 **Tip:** `envFrom` is convenient but be careful — all ConfigMap keys are injected as env vars, which could conflict with existing env vars.

#### As a Volume (File-Based Config)

```yaml
spec:
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  containers:
  - name: app
    image: my-app:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config         # Each key becomes a file here
```

With the config above, files are created at:
- `/etc/config/LOG_LEVEL`
- `/etc/config/DB_HOST`
- `/etc/config/app.yaml`
- `/etc/config/nginx.conf`

#### Mount Specific Keys as Files

```yaml
spec:
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.yaml
        path: application.yaml     # Rename the key to a specific path
      - key: nginx.conf
        path: conf.d/default.conf
  containers:
  - name: app
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app
```

### Updating ConfigMaps

```bash
kubectl edit configmap app-config

kubectl apply -f updated-configmap.yaml

# Patch a specific key
kubectl patch configmap app-config --patch '{"data":{"LOG_LEVEL":"debug"}}'
```

> 📝 **Note:** Pods do NOT automatically restart when ConfigMaps are updated. For environment variable injection, you must restart pods. For volume mounts, files update automatically (with ~1 minute delay). Use a configuration reloader (e.g., Reloader operator) for automatic pod restarts.

---

## Secrets

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Arbitrary user-defined data (most common) |
| `kubernetes.io/service-account-token` | ServiceAccount token |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/ssh-auth` | SSH credentials |

### Creating Secrets

#### From Literal Values

```bash
kubectl create secret generic db-secret \
  --from-literal=username=dbadmin \
  --from-literal=password='P@ssw0rd!'
```

#### From Files

```bash
kubectl create secret generic tls-secret \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key

# TLS type
kubectl create secret tls my-tls-secret \
  --cert=server.crt \
  --key=server.key
```

#### Docker Registry Secret

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=myregistry.azurecr.io \
  --docker-username=<sp-client-id> \
  --docker-password=<sp-client-secret>
```

#### Declarative YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque
data:
  # Values must be base64 encoded
  username: ZGJhZG1pbg==      # echo -n "dbadmin" | base64
  password: UEBzc1cwcmQh      # echo -n "P@ssw0rd!" | base64
```

```bash
# Encode a value for use in Secret YAML
echo -n "my-password" | base64

# Decode a value
echo "bXktcGFzc3dvcmQ=" | base64 --decode
```

> ⚠️ **Warning:** Base64 is NOT encryption — it's just encoding. Secrets are stored in etcd in base64 form. For real security, enable etcd encryption at rest or use Azure Key Vault CSI Driver (covered in Module 06).

### Using Secrets in Pods

#### As Environment Variables

```yaml
spec:
  containers:
  - name: app
    image: my-app:1.0
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

#### All Keys as Environment Variables

```yaml
spec:
  containers:
  - name: app
    envFrom:
    - secretRef:
        name: db-secret
```

#### As a Volume (File-Based)

```yaml
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400         # Read-only by owner
  containers:
  - name: app
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
```

Files created:
- `/etc/secrets/username`
- `/etc/secrets/password`

#### Image Pull Secret

To pull images from a private registry:

```yaml
spec:
  imagePullSecrets:
  - name: acr-secret
  containers:
  - name: app
    image: myregistry.azurecr.io/my-app:1.0
```

> 💡 **Tip:** For AKS, it's better to attach an ACR to your cluster using managed identity instead of image pull secrets:
> ```bash
> az aks update --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --attach-acr myregistry
> ```

---

## ConfigMap vs Secret Comparison

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| Data encoding | Plain text | Base64 |
| Encryption at rest | No (by default) | Optional |
| Suitable for | Config files, env vars, flags | Passwords, tokens, keys, certs |
| Size limit | 1 MiB | 1 MiB |
| Volume mount | Yes | Yes |
| Env injection | Yes | Yes |
| RBAC control | Standard | Standard (limit access!) |

---

## Best Practices

### For ConfigMaps

1. **Use version labels** to track changes: `version: "2024-01-15"`
2. **Separate configs by environment** — don't use `if env == prod` in configs
3. **Use volume mounts for file configs**, env injection for simple key-value
4. **Watch for pod restart requirements** after updates

### For Secrets

1. **Never commit secrets to Git** — use sealed secrets, Vault, or Key Vault CSI
2. **Use RBAC to restrict Secret access** — follow least privilege
3. **Enable etcd encryption** in production clusters
4. **Prefer Azure Key Vault CSI Driver** for enterprise workloads (Module 06)
5. **Rotate secrets regularly** and update references
6. **Use `stringData` in manifests** for readability (Kubernetes base64-encodes automatically):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:               # Plain text — Kubernetes encodes automatically
  username: dbadmin
  password: "P@ssw0rd!"
```

---

## Useful Commands

```bash
# List configmaps
kubectl get configmaps
kubectl get cm

# View configmap data
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config

# List secrets
kubectl get secrets

# View secret (base64 encoded)
kubectl get secret db-secret -o yaml

# Decode a secret value
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 --decode

# Delete
kubectl delete configmap app-config
kubectl delete secret db-secret
```

---

## Next Steps

Continue to [04-statefulsets.md](./04-statefulsets.md) to learn about stateful application deployment.
