# Kubernetes Production Engineering & Troubleshooting Playbook

A complete technical record and interview-ready case study covering CNI kernel network failure diagnostics, worker node recovery, and deployment scaling/rollout management.

---

## 1. Executive Summary & 1-Minute Elevator Pitch

> *"In a multi-node Kubernetes cluster, I resolved a critical CNI failure where `kubworker-node2` was scheduling pods into a persistent `ContainerCreating` state. Inspecting `kubectl describe pod` and container logs showed `FailedCreatePodSandbox: failed to load flannel subnet.env` because the `kube-flannel` DaemonSet was in `CrashLoopBackOff` due to missing `br_netfilter` kernel configurations. I loaded `br_netfilter`, enabled `bridge-nf-call-iptables`, `bridge-nf-call-ip6tables`, and `net.ipv4.ip_forward`, and persisted them across reboots via `/etc/modules-load.d/` and `/etc/sysctl.d/`. Once the pod was recycled, all 5 deployment replicas scheduled cleanly across both worker nodes. I then validated deployment operations by scaling to 15 replicas, executing rolling updates via `kubectl set image`, and tracking rollout state."*

---

## 2. Production Incident Case Study: CNI Network Failure & Node Recovery

### 2.1 The Symptom
* **Observed State:** When deploying `nginx-deploy` (5 replicas), only 3 pods scheduled on `kubworker-node1` reached `Running` state (3/5 Ready).
* **Failing Pods:** Pods scheduled on `kubworker-node2` (`nginx-deploy-5dcc99ff6-h9clr`, `v899g`) stayed permanently in `ContainerCreating`.

```bash
root@Kubmaster-node:~/kubernetes# kubectl get pods -o wide
NAME                          READY   STATUS              RESTARTS   AGE     IP           NODE
nginx-deploy-5dcc99ff6-5jgj4   1/1     Running             0          3m12s   10.244.1.6   kubworker-node1
nginx-deploy-5dcc99ff6-7l98k   1/1     Running             0          3m12s   10.244.1.8   kubworker-node1
nginx-deploy-5dcc99ff6-h9clr   0/1     ContainerCreating   0          3m12s   <none>       kubworker-node2
nginx-deploy-5dcc99ff6-knqzb   1/1     Running             0          3m12s   10.244.1.7   kubworker-node1
nginx-deploy-5dcc99ff6-v899g   0/1     ContainerCreating   0          3m12s   <none>       kubworker-node2
```

---

### 2.2 Root Cause Analysis (Layer by Layer)

#### Step 1: Inspect Failed Pod Events
```bash
kubectl describe pod nginx-deploy-5dcc99ff6-h9clr
```
* **Event Output:**
  ```text
  Warning  FailedCreatePodSandbox  6m26s  kubelet
  Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox:
  plugin type="flannel" failed (add): failed to load flannel 'subnet.env' file:
  open /run/flannel/subnet.env: no such file or directory
  ```

#### Step 2: Inspect CNI DaemonSet Status
```bash
kubectl get pods -n kube-flannel -o wide
```
* **Output:**
  ```text
  NAME                    READY   STATUS             RESTARTS        IP              NODE
  kube-flannel-ds-5fg9g   1/1     Running            3 (35h ago)     172.31.13.232   kubmaster-node
  kube-flannel-ds-7vhqk   0/1     CrashLoopBackOff   480 (41s ago)   172.31.12.72    kubworker-node2
  kube-flannel-ds-cp5vs   1/1     Running            4 (40m ago)     172.31.4.17     kubworker-node1
  ```

#### Step 3: Inspect Flannel Crash Logs
```bash
kubectl logs -n kube-flannel kube-flannel-ds-7vhqk --previous
```
* **Root Error Identified:**
  ```text
  E0829 02:28:24.412497 1 main.go:292] Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables: no such file or directory
  ```

---

### 2.3 Resolution & Permanent Fix

#### Step 1: Load Kernel Module on Worker Node (`kubworker-node2`)
```bash
# Check existing module state
lsmod | grep br_netfilter

# Load the required kernel bridge netfilter module
sudo modprobe br_netfilter
```

#### Step 2: Configure Required Kernel Parameters
```bash
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
sudo sysctl -w net.bridge.bridge-nf-call-ip6tables=1
sudo sysctl -w net.ipv4.ip_forward=1
```

#### Step 3: Persist Changes Across System Reboots
```bash
# 1. Persist kernel module loading
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 2. Persist sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# 3. Reload system settings immediately
sudo sysctl --system
```

#### Step 4: Recycle Flannel DaemonSet Pod (From Master)
```bash
kubectl delete pod -n kube-flannel kube-flannel-ds-7vhqk
```

#### Step 5: Verification of Cluster Recovery
```bash
root@Kubmaster-node:~/kubernetes# kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-deploy-5dcc99ff6-5jgj4   1/1     Running   0          22m   10.244.1.6   kubworker-node1
nginx-deploy-5dcc99ff6-7l98k   1/1     Running   0          22m   10.244.1.8   kubworker-node1
nginx-deploy-5dcc99ff6-h9clr   1/1     Running   0          22m   10.244.2.3   kubworker-node2
nginx-deploy-5dcc99ff6-knqzb   1/1     Running   0          22m   10.244.1.7   kubworker-node1
nginx-deploy-5dcc99ff6-v899g   1/1     Running   0          22m   10.244.2.2   kubworker-node2
```
* **Result:** `kubworker-node2` assigned pod IPs (`10.244.2.2`, `10.244.2.3`) successfully. All 5/5 pods transitioned to `Running`.

---

## 3. Side Incident: `connection refused: 6443` from Worker Node

### Diagnostic Scenario:
When attempting `kubectl delete pod` directly on `kubworker-node2`, the command failed:
```text
The connection to the server 172.31.12.72:6443 was refused - did you specify the right host or port?
```

### Explanation & Key Interview Takeaway:
* **The Root Cause:** `kubectl` is a client tool configured via `.kube/config`. On worker nodes, `kubectl` either lacked credentials or had an unconfigured/misconfigured `server:` endpoint pointing to the worker's own IP (`172.31.12.72:6443`) where no API server daemon runs.
* **Control Plane Validation:**
  ```bash
  # Check if kube-apiserver is active on master
  sudo crictl ps | grep kube-apiserver
  sudo ss -lntp | grep 6443

  # Test TCP reachability from worker to master
  nc -zv 172.31.13.232 6443
  ```
* **Architectural Rule:** Cluster administration commands should run from the master/control plane node (or workstation with valid admin kubeconfig pointing to master IP).

---

## 4. Workload Operations & Deployment Lifecycle

### 4.1 Deployment YAML Manifest (`nginx-deploy.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
        - name: nginx-container
          image: nginx:stable-otel
          ports:
            - containerPort: 80
```

---

### 4.2 Scaling Workloads Horizontally

```bash
# Scale up from 5 to 15 replicas
kubectl scale deployment nginx-deploy --replicas=15

# Verify pod distribution across worker nodes
kubectl get pods -l app=nginx-app -o wide

# Scale down to 1 replica
kubectl scale deployment nginx-deploy --replicas=1
```

---

### 4.3 Rolling Updates & Rollout Management

```bash
# 1. Update container image
kubectl set image deployment/nginx-deploy nginx-container=nginx:stable-otel

# 2. Track rollout status
kubectl rollout status deployment/nginx-deploy

# 3. View rollout history
kubectl rollout history deployment/nginx-deploy

# 4. Rollback to previous version if image fails (e.g., ImagePullBackOff / InvalidImageName)
kubectl rollout undo deployment/nginx-deploy
```

---

## 5. Master Troubleshooting Command Matrix

| Target Layer | Diagnostic Command | Expected Normal State |
| :--- | :--- | :--- |
| **Node State** | `kubectl get nodes -o wide` | Status = `Ready` |
| **Pod Status** | `kubectl get pods -o wide --all-namespaces` | Status = `Running`, Restarts stable |
| **Pod Events** | `kubectl describe pod <pod-name>` | Normal Scheduled, Pulled, Created, Started |
| **CNI Logs** | `kubectl logs -n kube-flannel <flannel-pod> --previous` | `backend: vxlan`, Subnet allocated |
| **Kernel Module** | `lsmod \| grep br_netfilter` | `br_netfilter` listed |
| **Sysctl Routing** | `sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward` | Values equal `1` |
| **API Listening** | `ss -lntp \| grep 6443` | `LISTEN` on 0.0.0.0:6443 |
| **Network Probe** | `nc -zv <MASTER_IP> 6443` | `open` / Connection succeeded |
| **Rollout State** | `kubectl rollout status deployment/<deploy-name>` | `deployment successfully rolled out` |
