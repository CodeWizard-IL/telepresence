# Lab 5: The "Deep Dive" :mag:

## Volume Mounts & Debugging

!!! info "Objective"
    Handle file-based dependencies (like SSL certs or ConfigMaps) and perform **live debugging** of cluster services from your IDE.

---

## Overview

Some services depend on **files** mounted into their pods — ConfigMaps, Secrets as files, SSL certificates, or custom configuration files. Telepresence can mount these remote volumes locally using `sshfs`, so your local code can read the same files the pod uses. Combined with an IDE debugger, this gives you the ultimate local development experience.

```mermaid
flowchart TD
    subgraph Cluster Pod
        V1["/etc/config/settings.json<br/>(ConfigMap)"]
        V2["/etc/ssl/certs/tls.crt<br/>(Secret)"]
        V3["/var/data/cache<br/>(PVC)"]
    end
    
    subgraph Local Machine
        TP["$TELEPRESENCE_ROOT"]
        TP --> LV1["$TELEPRESENCE_ROOT/etc/config/settings.json"]
        TP --> LV2["$TELEPRESENCE_ROOT/etc/ssl/certs/tls.crt"]
        TP --> LV3["$TELEPRESENCE_ROOT/var/data/cache"]
        LV1 --> IDE[IDE + Debugger]
        LV2 --> IDE
        LV3 --> IDE
    end
    
    V1 -.->|sshfs mount| LV1
    V2 -.->|sshfs mount| LV2
    V3 -.->|sshfs mount| LV3
    
    style TP fill:#FF9800,color:#fff
    style IDE fill:#4CAF50,color:#fff
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Telepresence | Connected to the cluster |
| `sshfs` | Installed on your local machine |
| A deployed service | With ConfigMap/Secret volume mounts |
| IDE with debugger | VS Code, GoLand, PyCharm, etc. |

### Install sshfs

=== "macOS"

    ```bash
    brew install macfuse
    brew install gromgit/fuse/sshfs-mac
    ```

    !!! warning
        macFUSE requires a system extension. You may need to allow it in **System Preferences > Privacy & Security** and reboot.

=== "Linux"

    ```bash
    sudo apt-get install sshfs
    ```

---

## Step 1: Deploy a Service with Volume Mounts

Create a deployment with a ConfigMap mounted as a file:

```yaml title="backend-with-volumes.yaml"
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  settings.json: |
    {
      "app_name": "Telepresence Lab",
      "version": "1.0.0",
      "features": {
        "dark_mode": true,
        "beta_features": false,
        "max_connections": 100
      },
      "endpoints": {
        "auth": "http://auth.default.svc.cluster.local",
        "cache": "http://redis.default.svc.cluster.local:6379"
      }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: tls-certs
type: Opaque
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    (sample certificate data)
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    (sample key data)
    -----END PRIVATE KEY-----
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
        - name: certs
          mountPath: /etc/ssl/app-certs
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: app-settings
      - name: certs
        secret:
          secretName: tls-certs
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
  type: ClusterIP
```

Apply it:

```bash
kubectl apply -f backend-with-volumes.yaml
```

Verify the volumes are mounted:

```bash
kubectl exec deploy/backend -- ls /etc/config/
kubectl exec deploy/backend -- cat /etc/config/settings.json
```

---

## Step 2: Intercept with Volume Mounts

Create an intercept with the `--mount` flag:

```bash
telepresence intercept backend \
  --port 8080:8080 \
  --mount=true
```

??? example "Expected Output"
    ```
    ✔ Intercepted
       Using Deployment backend
          Intercept name    : backend
          State             : ACTIVE
          Workload kind     : Deployment
          Intercepting      : all TCP connections
              8080 -> 8080 TCP
          Volume Mount Point: /tmp/telfs-xxxxxxx
    ```

!!! note "TELEPRESENCE_ROOT"
    Telepresence sets the `$TELEPRESENCE_ROOT` environment variable pointing to the mount directory. All remote filesystem paths are available under this prefix.

---

## Tasks

### Task 1: Explore the Mounted Volumes

List the mounted files:

```bash
echo $TELEPRESENCE_ROOT
ls $TELEPRESENCE_ROOT/etc/config/
cat $TELEPRESENCE_ROOT/etc/config/settings.json
```

??? example "Expected Output"
    ```json
    {
      "app_name": "Telepresence Lab",
      "version": "1.0.0",
      "features": {
        "dark_mode": true,
        "beta_features": false,
        "max_connections": 100
      },
      "endpoints": {
        "auth": "http://auth.default.svc.cluster.local",
        "cache": "http://redis.default.svc.cluster.local:6379"
      }
    }
    ```

Also check the mounted certificates:

```bash
ls $TELEPRESENCE_ROOT/etc/ssl/app-certs/
```

---

### Task 2: Use Mounted Files in Your Local Code

Create an application that reads the remote ConfigMap:

```python title="app_with_config.py"
import json
import os
from http.server import HTTPServer, BaseHTTPRequestHandler

# Read config from the mounted volume
telepresence_root = os.getenv("TELEPRESENCE_ROOT", "")
config_path = os.path.join(telepresence_root, "etc/config/settings.json")

with open(config_path) as f:
    config = json.load(f)

print(f"Loaded config: {config['app_name']} v{config['version']}")

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        response = {
            "app": config["app_name"],
            "version": config["version"],
            "features": config["features"],
            "source": "local-with-remote-config"
        }
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps(response, indent=2).encode())

if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", 8080), Handler)
    print("Server running on port 8080 with remote config...")
    server.serve_forever()
```

Run it:

```bash
python3 app_with_config.py
```

Test it:

```bash
curl http://backend.default:8080
```

---

### Task 3: Attach a Debugger (VS Code)

Create a VS Code debug configuration:

```json title=".vscode/launch.json"
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug with Telepresence",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/app_with_config.py",
      "envFile": "${workspaceFolder}/k8s-spec.env",
      "env": {
        "TELEPRESENCE_ROOT": "${env:TELEPRESENCE_ROOT}"
      }
    }
  ]
}
```

Steps:

1. Set a breakpoint in `app_with_config.py` on the `do_GET` method.
2. Press ++f5++ to start debugging.
3. From another terminal, trigger a request:

    ```bash
    curl http://backend.default:8080
    ```

4. Watch the execution **pause at your breakpoint** in VS Code.

!!! success "Key Insight"
    You are debugging a **cluster service** as if it were a local application, with full access to remote config files and the cluster network.

---

### Task 4: Inspect Remote Filesystem Structure

Explore the full remote filesystem available to your pod:

```bash
# See what's available
find $TELEPRESENCE_ROOT -maxdepth 3 -type f | head -30
```

This gives you visibility into:

- Mounted ConfigMaps and Secrets
- Service account tokens
- Kubernetes downward API files
- Any PersistentVolumeClaim data

---

## Cleanup

Leave the intercept:

```bash
telepresence leave backend
```

Remove test resources:

```bash
kubectl delete -f backend-with-volumes.yaml
```

Disconnect from the cluster:

```bash
telepresence quit
```

---

## Outcome

!!! success "What You Learned"
    - [x] Telepresence can mount remote pod volumes locally via `sshfs`
    - [x] ConfigMaps, Secrets, and PVCs are accessible under `$TELEPRESENCE_ROOT`
    - [x] Your local code can read the same files as the remote pod
    - [x] You can attach an IDE debugger and set breakpoints on intercepted traffic
    - [x] You have mastered debugging complex, stateful microservices as if they were monolithic local apps
