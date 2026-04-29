## Reference Manifest

The following manifest serves as a reference for the examples used throughout this document. It defines a Pod named `demo-pod` running nginx within the `demo` namespace. The namespace itself is created imperatively beforehand using `kubectl create namespace demo`.

```yaml
# manifest.yml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
```

<br><br>

## Command Reference

| Command | Description | Example |
|---|---|---|
| `kubectl get pods` | Lists all Pods in the current namespace. Displays each Pod's name, readiness state, status, restart count, and age. | — |
| `kubectl get pods -n <namespace_name>` | Lists all Pods within the specified namespace. Equivalent to `kubectl get pods` but scoped to a particular namespace. | `kubectl get pods -n demo` |
| `kubectl get namespaces` | Lists all namespaces in the cluster, along with their status and age. | — |
| `kubectl create namespace <namespace_name>` | Creates a new namespace imperatively without a manifest file. | kubectl create namespace demo |
| `kubectl run <pod_name> --image=<image_name>` | Creates and starts a new Pod with the specified name using the given container image. This is an imperative command and does not use a manifest file. | `kubectl run demo-pod --image=nginx` |
| `kubectl apply -f <file_name>` | Creates or updates Kubernetes resources defined in a YAML or JSON manifest file. If the resource does not exist, it is created; if it does, it is updated to match the file. | `kubectl apply -f pod.yml` |
| `kubectl describe pod <pod_name>` | Displays detailed information about a specific Pod, including its events, container states, resource requests, environment variables, volume mounts, and scheduling information. | `kubectl describe pod demo-pod` |
| `kubectl describe ns <namespace_name>` | Displays detailed information about a specific namespace, including its labels, annotations, status, and any resource quotas or limit ranges applied to it. | `kubectl describe ns demo` |
| `kubectl delete pod <pod_name>` | Deletes a specific Pod by name from the current namespace. The Pod is terminated and removed from the cluster. | `kubectl delete pod demo-pod` |
| `kubectl delete -f <file_name>` | Deletes all Kubernetes resources defined in the specified manifest file. This is the declarative equivalent of deleting resources by name. | `kubectl delete -f pod.yml` |
| `kubectl port-forward <pod_name> <local_port>:<pod_port>` | Forwards a local port on the host machine to a port on the specified Pod. Useful for accessing a Pod's service locally without exposing it via a Service resource. | `kubectl port-forward demo-pod 2224:80` |
| `kubectl top pods` | Displays real-time CPU and memory usage for all Pods in the current namespace. Requires the Metrics Server to be installed in the cluster. | — |
| `kubectl top pods -A` | Displays real-time CPU and memory usage for all Pods across all namespaces. The `-A` flag is shorthand for `--all-namespaces`. | — |
| `kubectl top nodes` | Displays real-time CPU and memory usage for each node in the cluster. Useful for identifying resource pressure at the infrastructure level. | — |
| `kubectl exec -it <pod_name> -- <command>` | Executes a command inside a running container in the specified Pod. The `-i` flag keeps standard input open for interaction. The `-t` flag allocates a pseudo-terminal for an interactive shell experience. The `--` separator distinguishes kubectl flags from the command being run inside the container. | `kubectl exec -it demo-pod -- sh` |
| `kubectl cp <source_file> <pod_name>:<destination_path>` | Copies a file from the local filesystem into a running container at the specified path. The destination path is relative to the container's filesystem root. | `kubectl cp nginx.conf demo-pod:/etc/nginx/nginx.conf` |
| `vim <file_name>` | Opens the specified file in the Vim text editor for editing. Commonly used to create or modify Kubernetes manifest files before applying them. | `vim pod.yml` |

<br><br>

## Kubernetes Probes

Probes are mechanisms used by the kubelet to periodically check the state of a container inside a Pod. They determine whether a container is healthy, ready to serve traffic, or still initializing, and allow Kubernetes to take automated corrective action when a container is not behaving as expected.

1. **Liveness Probe:** Answers the question: "Is this container still functioning correctly?" If the probe fails repeatedly, Kubernetes assumes the container is stuck or in a broken state and restarts it. This is useful for detecting deadlocks or processes that are running but no longer making progress.

2. **Readiness Probe:** Answers the question: "Is this container ready to receive traffic?" If the probe fails, the container is not restarted. Instead, it is temporarily removed from the Service's endpoints so it stops receiving requests until it passes again. This is useful during startup, warm-up periods, or transient failures.

3. **Startup Probe:** Designed for containers with slow initialization. It instructs Kubernetes to delay liveness and readiness checks until the application has fully started. Without a startup probe, slow-starting containers risk being killed prematurely by the liveness probe before they have had time to initialize.

### Probe Implementation Methods

Probes can be implemented in three ways, depending on what the container exposes:

- **httpGet** — Makes an HTTP GET request to a specified path and port. A response in the 200–399 range is considered a success.
- **exec** — Runs a command inside the container. An exit code of `0` is considered a success.
- **tcpSocket** — Attempts to open a TCP connection to a specified port. A successful connection is considered a success.

```yaml
# exec probe example
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10

# tcpSocket probe example
livenessProbe:
  tcpSocket:
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Complete Pod Manifest with All Three Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
        timeoutSeconds: 1
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /health
          port: 80
        initialDelaySeconds: 2
        periodSeconds: 5
```

In this configuration, the startup probe runs first and gives the container up to 300 seconds (30 failures × 10 second intervals) to initialize. Once it passes, the liveness probe begins checking the root path every 10 seconds and restarts the container after 3 consecutive failures. The readiness probe checks `/health` every 5 seconds and removes the Pod from Service endpoints if it fails, without triggering a restart.

<br><br>

## Volumes

A container's filesystem is ephemeral by default. When a container restarts, all data written to its filesystem is lost. Volumes solve this by providing storage that exists independently of the container's lifecycle and can be shared between containers within the same Pod. A volume is defined at the Pod level and mounted into one or more containers within that Pod. The lifetime of the volume is tied to the Pod unless an external storage backend is used, in which case data can persist beyond the Pod's lifecycle entirely. Common Volume Types include:

1. **emptyDir** — Created when the Pod is assigned to a node and deleted when the Pod terminates. Useful for temporary scratch space or sharing data between containers in the same Pod. Data is lost when the Pod is deleted or rescheduled to a different node. It is not suitable for data that must persist across Pod restarts.

```yaml
volumes:
  - name: shared-data
    emptyDir: {}
```

2. **hostPath** — Mounts a file or directory from the host node's filesystem into the Pod. Useful for accessing node-level resources, but tightly couples the Pod to a specific node, which is generally discouraged in production. These volumes bypass Kubernetes storage abstractions and can pose a security risk if containers are allowed to mount arbitrary host paths. Avoid in multi-tenant clusters.

```yaml
volumes:
  - name: host-logs
    hostPath:
      path: /var/log
      type: Directory
```

3. **persistentVolumeClaim** — Mounts a PersistentVolume provisioned through a PersistentVolumeClaim (PVC). This is the standard approach for durable, production-grade storage. The actual storage backend (cloud disk, NFS, etc.) is abstracted away from the Pod definition.

```yaml
volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### Full Pod Example with an emptyDir Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
    - name: content-writer
      image: busybox
      command: ["/bin/sh", "-c", "echo 'Hello' > /data/index.html && sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
  volumes:
    - name: shared-data
      emptyDir: {}
```

In this example, two containers share the same `emptyDir` volume. The `content-writer` container writes a file to `/data`, and the `nginx` container serves that file from `/usr/share/nginx/html`. Both paths point to the same underlying volume.

<br><br>

## ConfigMaps

A ConfigMap is a Kubernetes resource used to store non-sensitive configuration data as key-value pairs. It decouples configuration from container images, allowing the same image to be used across different environments by simply changing the ConfigMap rather than rebuilding the image. ConfigMaps are stored in plain text within etcd. They are not encrypted and are not intended for sensitive data. Any user or process with read access to the namespace can retrieve a ConfigMap's contents. Pods do not automatically pick up changes to a ConfigMap when it is consumed as environment variables. The Pod must be restarted for changes to take effect. When consumed as a mounted volume, Kubernetes will eventually propagate updates to the file, though there may be a short delay.

### Creating a ConfigMap

**Imperatively:**

```bash
kubectl create configmap <configmap_name> --from-literal=<key>=<value>
kubectl create configmap <configmap_name> --from-file=<file_name>
```

**Declaratively:**

```yaml
# configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:
  APP_ENV: production
  APP_PORT: "8080"
  LOG_LEVEL: info
```

### Using a ConfigMap in a Pod Manifest

**As environment variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      envFrom:
        - configMapRef:
            name: app-config
```

**As a mounted file:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```


<br><br>

## Secrets

A Secret is a Kubernetes resource designed to hold sensitive data such as passwords, API keys, TLS certificates, and tokens. Structurally, Secrets are similar to ConfigMaps, but Kubernetes applies additional handling around them to limit accidental exposure. Secret values are stored as Base64-encoded strings. It is critical to understand that Base64 encoding is not encryption. It is simply an encoding scheme, and any Base64-encoded value can be trivially decoded. This means that a Secret, by itself, does not guarantee confidentiality.

- Secrets are only sent to nodes that have Pods explicitly requesting them, reducing unnecessary exposure across the cluster.
- They are stored separately from ConfigMaps in etcd, which allows cluster administrators to apply stricter RBAC policies and audit logging specifically to Secrets.
- Kubernetes supports enabling encryption at rest for Secrets via an `EncryptionConfiguration` resource applied to the API server. This is not enabled by default and must be configured explicitly.
- Access to Secrets can and should be restricted using Role-Based Access Control (RBAC). Without RBAC rules in place, any user with namespace access can read all Secrets in that namespace.
- Secrets are never automatically rotated. Rotation must be handled externally, either manually or through a secrets management tool such as HashiCorp Vault or AWS Secrets Manager.

### Creating a Secret

**Imperatively:**

```bash
kubectl create secret generic <secret_name> --from-literal=<key>=<value>
kubectl create secret generic <secret_name> --from-file=<file_name>
```

**Declaratively:**

Values must be Base64-encoded when defined in a manifest.

```bash
echo -n 'mysecretpassword' | base64
# Output: bXlzZWNyZXRwYXNzd29yZA==
```

```yaml
# secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
data:
  DB_PASSWORD: bXlzZWNyZXRwYXNzd29yZA==
  API_KEY: c29tZWFwaWtleQ==
```

The `type: Opaque` field indicates a generic, arbitrary Secret. Kubernetes also provides built-in types for specific use cases, such as `kubernetes.io/tls` for TLS certificates and `kubernetes.io/dockerconfigjson` for image pull credentials.

### Using a Secret in a Pod Manifest

**As environment variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      envFrom:
        - secretRef:
            name: app-secret
```

**As a mounted file:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
```

Each key in the Secret becomes a file inside `/etc/secrets`, with the decoded value as its content. Marking the mount as `readOnly: true` is good practice to prevent the container from accidentally modifying the Secret data.

Enabling immutability on a Secret prevents its data from being changed after creation, which protects against accidental modification. This is done by setting `immutable: true` in the Secret manifest.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
immutable: true
type: Opaque
data:
  DB_PASSWORD: bXlzZWNyZXRwYXNzd29yZA==
```

<br><br>

## Command Reference for Volumes, ConfigMap and Secrets

| Command | Description | Example |
|---|---|---|
| `kubectl create configmap <configmap_name> --from-literal=<key>=<value>` | Creates a ConfigMap imperatively using a key-value pair provided directly in the command. Multiple `--from-literal` flags can be chained. | `kubectl create configmap app-config --from-literal=APP_ENV=production` |
| `kubectl create configmap <configmap_name> --from-file=<file_name>` | Creates a ConfigMap from a file. The filename becomes the key and the file contents become the value. | `kubectl create configmap app-config --from-file=config.properties` |
| `kubectl get configmaps` | Lists all ConfigMaps in the current namespace. | — |
| `kubectl get configmaps -n <namespace_name>` | Lists all ConfigMaps in the specified namespace. | `kubectl get configmaps -n demo` |
| `kubectl describe configmap <configmap_name>` | Displays the full contents and metadata of a specific ConfigMap. | `kubectl describe configmap app-config` |
| `kubectl delete configmap <configmap_name>` | Deletes the specified ConfigMap from the cluster. | `kubectl delete configmap app-config` |
| `kubectl create secret generic <secret_name> --from-literal=<key>=<value>` | Creates a generic Secret imperatively using a key-value pair. The value is automatically Base64-encoded by Kubernetes. | `kubectl create secret generic app-secret --from-literal=DB_PASSWORD=mysecretpassword` |
| `kubectl create secret generic <secret_name> --from-file=<file_name>` | Creates a generic Secret from a file. The filename becomes the key and the file contents become the Base64-encoded value. | `kubectl create secret generic app-secret --from-file=credentials.txt` |
| `kubectl get secrets` | Lists all Secrets in the current namespace. Values are not displayed. | — |
| `kubectl get secrets -n <namespace_name>` | Lists all Secrets in the specified namespace. | `kubectl get secrets -n demo` |
| `kubectl describe secret <secret_name>` | Displays metadata and key names of a Secret. Values are intentionally hidden and shown only as their byte length. | `kubectl describe secret app-secret` |
| `kubectl get secret <secret_name> -o jsonpath='{.data.<key>}'` | Retrieves the Base64-encoded value of a specific key from a Secret. Pipe the output to `base64 --decode` to read the plain text value. | `kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' \| base64 --decode` |
| `kubectl delete secret <secret_name>` | Deletes the specified Secret from the cluster. | `kubectl delete secret app-secret` |
| `kubectl get persistentvolumes` | Lists all PersistentVolumes in the cluster. PersistentVolumes are cluster-scoped, not namespace-scoped. | — |
| `kubectl get persistentvolumeclaims -n <namespace_name>` | Lists all PersistentVolumeClaims in the specified namespace. | `kubectl get persistentvolumeclaims -n demo` |
| `kubectl describe persistentvolumeclaim <pvc_name>` | Displays detailed information about a PersistentVolumeClaim, including its bound status, capacity, and access modes. | `kubectl describe persistentvolumeclaim my-pvc` |
