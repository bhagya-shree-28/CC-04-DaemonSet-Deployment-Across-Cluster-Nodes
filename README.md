# DaemonSet Deployment Across Cluster Nodes

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Objectives](#2-objectives)
3. [Background Concepts](#3-background-concepts)
4. [Lab Environment & Architecture](#4-lab-environment--architecture)
5. [Prerequisites](#5-prerequisites)
6. [Implementation Steps](#6-implementation-steps)
7. [Results Summary](#7-results-summary)
8. [Key Observations & Learnings](#8-key-observations--learnings)
9. [Optional Extension: Node Added / Removed](#9-optional-extension-node-added--removed)
10. [Cleanup](#10-cleanup)
11. [Conclusion](#11-conclusion)

---

## 1. Problem Statement

Deploy a node-level agent, such as a log collector, as a Kubernetes **DaemonSet** so that exactly one pod runs on every worker node. Observe what happens when nodes are added or restricted.


---

## 2. Objectives

- Write and deploy a DaemonSet manifest running a **Fluent Bit** log collector.
- Verify that Kubernetes schedules **exactly one pod per worker node** automatically.
- Restrict the DaemonSet to selected nodes using a **node label + `nodeSelector`**.
- Observe how a **taint** (`NoExecute`) keeps DaemonSet pods away from a node, and how a **toleration** lets them back in.
- Observe DaemonSet behaviour when a node is **cordoned** (marked unschedulable) and uncordoned.
- Record the DaemonSet status (`DESIRED / CURRENT / READY / UP-TO-DATE / AVAILABLE`) at every stage.

---

## 3. Background Concepts

### 3.1 DaemonSet
A DaemonSet is a Kubernetes controller that ensures a copy of a pod runs on all (or a selected subset of) nodes. When a node joins the cluster a pod is added to it; when a node is removed the pod is garbage collected. Deleting the DaemonSet deletes the pods it created.

### 3.2 nodeSelector
A simple constraint in the pod template: the pod is only scheduled on nodes whose labels match. Used here as `monitoring: enabled`.

### 3.3 Taints and Tolerations
- A **taint** on a node repels pods that do not tolerate it.
- A **toleration** on a pod allows it to be scheduled on (and stay on) a node with a matching taint.
- Taint effects:

| Effect | Behaviour |
|---|---|
| `NoSchedule` | New pods without a matching toleration are not scheduled on the node. |
| `PreferNoSchedule` | The scheduler tries to avoid the node but it is not guaranteed. |
| `NoExecute` | New pods are not scheduled **and existing pods without a toleration are evicted**. |

Here the taint is `dedicated=gpu:NoExecute`.

### 3.4 Cordon / Uncordon
`kubectl cordon <node>` marks a node unschedulable (`SchedulingDisabled`) so normal workloads stop landing on it. Existing pods keep running. `kubectl uncordon` reverses it.

### 3.5 Control-plane node
In a kind cluster the control-plane node carries the taint `node-role.kubernetes.io/control-plane:NoSchedule`, which our DaemonSet does not tolerate. That is why the DaemonSet targets only the **two worker nodes** (`DESIRED = 2`) and not all three nodes.

---

## 4. Lab Environment & Architecture

| Component | Details |
|---|---|
| Cluster | kind (Kubernetes in Docker), 1 control-plane + 2 workers |
| Nodes | `kind-control-plane`, `kind-worker`, `kind-worker2` |
| Kubernetes version | v1.37.0 (as reported by `kubectl get nodes`) |
| Host shell | Windows PowerShell |
| Working directory | `C:\Users\anany\k8s-lab` |
| DaemonSet name | `log-collector` |
| Container image | `fluent/fluent-bit:latest` |
| Resources | requests: `cpu: 50m`, `memory: 50Mi`; limit: `memory: 100Mi` |
| Manifest file | `log-agent-ds.yaml` |

---

## 5. Prerequisites

- Docker Desktop running
- `kind` and `kubectl` installed
- A running multi-node kind cluster (1 control-plane, 2 workers)

Verify the cluster before starting:

```powershell
kubectl get nodes
```

---

## 6. Implementation Steps

All manifests are written with PowerShell here-strings (`@" ... "@`) piped to `Out-File -Encoding utf8`.

### Step 1 — Deploy the baseline DaemonSet (one pod per node)

**Manifest — `log-agent-ds.yaml`**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        resources:
          limits:
            memory: 100Mi
          requests:
            cpu: 50m
            memory: 50Mi
```

**Commands**

```powershell
kubectl apply -f log-agent-ds.yaml
kubectl get daemonset
kubectl get pods -o wide
```

**Output (summarised)**

Right after `apply` (pods still pulling the image):

```
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
log-collector   2         2         0       2            0           <none>          10s

NAME                  READY   STATUS              NODE
log-collector-986hh   0/1     ContainerCreating   kind-worker2
log-collector-wr6jc   0/1     ContainerCreating   kind-worker
```

After the pods start:

```
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
log-collector   2         2         2       2            2           <none>          82s

NAME                  READY   STATUS    IP            NODE
log-collector-986hh   1/1     Running   10.244.2.19   kind-worker2
log-collector-wr6jc   1/1     Running   10.244.1.20   kind-worker
```


<img width="954" height="971" alt="WhatsApp Image 2026-11-14 at 2 08 23 PM" src="https://github.com/user-attachments/assets/ed625dbb-8011-46cb-aace-e397c1c5e0ad" />






**Observation:** `DESIRED = 2` — one pod was created on each worker node automatically, with no replica count specified. The control-plane node was skipped because of its `NoSchedule` taint.

---

### Step 2 — Restrict the DaemonSet with a label and `nodeSelector`

Label only `kind-worker`, then add a `nodeSelector` to the pod template.

**Commands**

```powershell
kubectl label node kind-worker monitoring=enabled
```

**Updated manifest (only the `spec.template.spec` changes)**

```yaml
    spec:
      nodeSelector:
        monitoring: enabled
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        resources:
          limits:
            memory: 100Mi
          requests:
            cpu: 50m
            memory: 50Mi
```

```powershell
kubectl apply -f log-agent-ds.yaml
kubectl get daemonset
kubectl get pods -o wide
```

**Output (summarised)**

```
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR        AGE
log-collector   1         1         1       1            1           monitoring=enabled   2m57s

NAME                  READY   STATUS    IP            NODE
log-collector-m5mmp   1/1     Running   10.244.1.21   kind-worker
```


<img width="956" height="820" alt="WhatsApp Image 2026-11-14 at 2 08 34 PM" src="https://github.com/user-attachments/assets/ae1cd106-3730-4fb2-ab43-0b1beeb9eb65" />


**Observation:** `DESIRED` dropped from 2 to 1. The pod on `kind-worker2` was removed because that node does not carry the `monitoring=enabled` label. Only the labelled node runs the agent, and the `NODE SELECTOR` column now shows the constraint.

---

### Step 3 — Taint a node (no toleration)

Remove the label, taint `kind-worker2`, and re-apply the manifest **without** a `nodeSelector` or toleration.

**Commands**

```powershell
kubectl label node kind-worker monitoring-
kubectl taint nodes kind-worker2 dedicated=gpu:NoExecute
```

**Manifest:** same as Step 1 (no `nodeSelector`, no `tolerations`).

```powershell
kubectl apply -f log-agent-ds.yaml
kubectl get daemonset
kubectl get pods -o wide
```

**Output (summarised)**

```
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
log-collector   1         1         0       1            0           <none>          4m34s

NAME                  READY   STATUS              NODE
log-collector-vqn9q   0/1     ContainerCreating   kind-worker
```


<img width="950" height="804" alt="WhatsApp Image 2026-11-14 at 2 11 41 PM" src="https://github.com/user-attachments/assets/670cbf41-2f68-47ad-a604-9357ddeee618" />


**Observation:** Even though the `nodeSelector` was removed, `DESIRED` stays at 1. `kind-worker2` is tainted with `dedicated=gpu:NoExecute` and the DaemonSet pod has no matching toleration, so it is **not scheduled there**. The only pod runs on `kind-worker`.

---

### Step 4 — Add a toleration so the DaemonSet can run on the tainted node

**Updated manifest (toleration added)**

```yaml
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "gpu"
        effect: "NoExecute"
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        resources:
          limits:
            memory: 100Mi
          requests:
            cpu: 50m
            memory: 50Mi
```

```powershell
kubectl apply -f log-agent-ds.yaml
kubectl get daemonset
kubectl get pods -o wide
kubectl get pods -o wide     # repeat after a few seconds
```

**Output (summarised)**

Immediately after apply (rolling update in progress):

```
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
log-collector   2         2         1       1            1           <none>          2m41s

NAME                  READY   STATUS              NODE
log-collector-hvpz5   1/1     Running             kind-worker
log-collector-k546j   0/1     ContainerCreating   kind-worker2
```

A few seconds later:

```
NAME                  READY   STATUS    IP            NODE
log-collector-k546j   1/1     Running   10.244.2.22   kind-worker2
log-collector-p82sn   1/1     Running   10.244.1.24   kind-worker
```


<img width="1505" height="869" alt="WhatsApp Image 2026-11-14 at 2 12 51 PM" src="https://github.com/user-attachments/assets/726bc778-6ae7-4c8a-a4f3-015f2ca76b9a" />



**Observation:**
- `DESIRED` went back from 1 to 2: the DaemonSet now **tolerates** the `dedicated=gpu:NoExecute` taint, so a pod was created on the tainted `kind-worker2`.
- The pod on `kind-worker` was also **replaced** (`hvpz5` → `p82sn`) because changing the pod template triggers a **RollingUpdate** of the DaemonSet.

---

### Step 5 — Remove the taint

**Command**

```powershell
kubectl taint nodes kind-worker2 dedicated=gpu:NoExecute-
kubectl get pods -o wide
```

**Output (summarised)**

```
node/kind-worker2 untainted

NAME                  READY   STATUS    NODE
log-collector-c8czs   1/1     Running   kind-worker2
log-collector-vqn9q   1/1     Running   kind-worker
```

**Observation:** With the taint removed, both workers are eligible again and each runs one pod. The trailing `-` in the command is what removes a taint.

---

### Step 6 — Cordon a node and observe DaemonSet behaviour

**Commands**

```powershell
kubectl cordon kind-worker2
kubectl delete pod -l app=log-collector --field-selector spec.nodeName=kind-worker2
kubectl get nodes
kubectl get pods -o wide
```

**Output (summarised)**

```
node/kind-worker2 cordoned
pod "log-collector-c8czs" deleted from default namespace

NAME                 STATUS                     ROLES           AGE   VERSION
kind-control-plane   Ready                      control-plane   11d   v1.37.0
kind-worker          Ready                      <none>          11d   v1.37.0
kind-worker2         Ready,SchedulingDisabled   <none>          11d   v1.37.0

NAME                  READY   STATUS    IP            NODE
log-collector-vqn9q   1/1     Running   10.244.1.22   kind-worker
log-collector-wmpvv   1/1     Running   10.244.2.21   kind-worker2
```


<img width="958" height="534" alt="WhatsApp Image 2026-11-14 at 2 14 33 PM" src="https://github.com/user-attachments/assets/7279c2ef-ef1a-4793-92d2-1ad13fcbbbc9" />



**Observation:** Although `kind-worker2` is `SchedulingDisabled`, the DaemonSet controller **immediately recreated** the deleted pod (`c8czs` → `wmpvv`) on that same node. This is because DaemonSet pods automatically receive a toleration for the `node.kubernetes.io/unschedulable:NoSchedule` taint that a cordon adds, so cordoning does not stop node-level agents such as log collectors.

---

### Step 7 — Uncordon and verify the final state

**Commands**

```powershell
kubectl uncordon kind-worker2
kubectl get nodes
kubectl get daemonset
kubectl get pods -o wide
```

**Output (summarised)**

```
node/kind-worker2 uncordoned

NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
log-collector   2         2         2       2            2           <none>          10m

NAME                  READY   STATUS    IP            NODE
log-collector-vqn9q   1/1     Running   10.244.1.22   kind-worker
log-collector-wmpvv   1/1     Running   10.244.2.21   kind-worker2
```


<img width="943" height="371" alt="WhatsApp Image 2026-11-14 at 2 16 14 PM" src="https://github.com/user-attachments/assets/c077ab24-1864-4af7-a88b-8d2a2bc8d6ab" />


**Observation:** All nodes are `Ready`, and the DaemonSet is fully healthy: 2 desired, 2 current, 2 ready, one pod on each worker.

---

###  Command Reference

All commands used in this experiment and what each one does:

| Command | Purpose | Used in |
|---|---|---|
| `@" ... "@ \| Out-File -Encoding utf8 log-agent-ds.yaml` | PowerShell here-string that writes the YAML manifest to a file with UTF-8 encoding | Steps 1-4 |
| `kubectl get nodes` | Lists cluster nodes with their status (`Ready`, `SchedulingDisabled`), roles and version | Prerequisites, Steps 6-7 |
| `kubectl apply -f log-agent-ds.yaml` | Creates the DaemonSet, or updates it if it already exists (`created` / `configured`) | Steps 1-4 |
| `kubectl get daemonset` | Shows `DESIRED`, `CURRENT`, `READY`, `UP-TO-DATE`, `AVAILABLE` and `NODE SELECTOR` for the DaemonSet | Steps 1-4, 7 |
| `kubectl get pods -o wide` | Lists pods with extra detail: pod IP and the node each pod runs on, which is how one pod per node is verified | Steps 1-7 |
| `kubectl delete deployment web` | Deletes the earlier `web` Deployment so only DaemonSet pods remain in the output | Step 1 |
| `kubectl label node kind-worker monitoring=enabled` | Adds a label to a node so a `nodeSelector` can match it | Step 2 |
| `kubectl label node kind-worker monitoring-` | Removes the label from the node (the trailing `-` means remove) | Step 3 |
| `kubectl taint nodes kind-worker2 dedicated=gpu:NoExecute` | Taints a node so pods without a matching toleration are not scheduled there, and existing ones are evicted | Step 3 |
| `kubectl taint nodes kind-worker2 dedicated=gpu:NoExecute-` | Removes the taint from the node | Step 5 |
| `kubectl cordon kind-worker2` | Marks the node unschedulable (`SchedulingDisabled`) for normal workloads; running pods are not touched | Step 6 |
| `kubectl delete pod -l app=log-collector --field-selector spec.nodeName=kind-worker2` | Deletes only the `log-collector` pod on `kind-worker2` (label filter plus node filter) to test whether the DaemonSet recreates it | Step 6 |
| `kubectl uncordon kind-worker2` | Makes the node schedulable again | Step 7 |

---

## 7. Results Summary

| # | Scenario | Node configuration | `DESIRED` | Pods running on | Result |
|---|---|---|---|---|---|
| 1 | Baseline DaemonSet | No label, no taint | 2 | `kind-worker`, `kind-worker2` | One pod per worker node |
| 2 | `nodeSelector: monitoring=enabled` | Only `kind-worker` labelled | 1 | `kind-worker` | Pod removed from unlabelled node |
| 3 | Taint, no toleration | `kind-worker2` tainted `dedicated=gpu:NoExecute` | 1 | `kind-worker` | Tainted node excluded |
| 4 | Taint + toleration | `kind-worker2` tainted, DaemonSet tolerates | 2 | `kind-worker`, `kind-worker2` | Tainted node included again |
| 5 | Taint removed | No taint | 2 | `kind-worker`, `kind-worker2` | Both nodes eligible |
| 6 | Node cordoned | `kind-worker2` `SchedulingDisabled` | 2 | `kind-worker`, `kind-worker2` | Pod recreated despite cordon |
| 7 | Node uncordoned | All nodes `Ready` | 2 | `kind-worker`, `kind-worker2` | Final healthy state (2/2/2/2/2) |

---

## 8. Key Observations & Learnings

1. **No replica count needed.** The DaemonSet controller derives the number of pods from the number of eligible nodes (`DESIRED` follows the node count, not a number we set).
2. **Control-plane nodes are skipped by default** because of their `NoSchedule` taint, unless a toleration is added.
3. **`nodeSelector` narrows the target set.** Labelling a node is all it takes to bring it into or out of scope, with no pod-level change.
4. **Taints repel, tolerations admit.** A `NoExecute` taint keeps (and evicts) pods without a matching toleration; adding the toleration to the pod template brings the DaemonSet back to that node.
5. **Template changes trigger a rolling update.** Editing the pod template (adding a toleration or `nodeSelector`) replaced existing pods, visible as new pod names.
6. **Cordon does not block DaemonSet pods.** The controller adds a built-in toleration for the unschedulable taint, so a cordoned node still gets its node-level agent.
7. **Self-healing.** Deleting a DaemonSet pod makes the controller recreate it on the same node within seconds.

---

## 9. Optional Extension: Node Added / Removed

The problem statement also asks to observe what happens when nodes are **added or removed**. The steps above cover restricting nodes (labels, taints, cordon). To demonstrate node membership changes directly:

**Node removed**

```powershell
kubectl delete node kind-worker2
kubectl get pods -o wide        # the pod on kind-worker2 is garbage collected
```

**Node added**

kind cannot add a worker to an existing cluster declaratively, so recreate the cluster with an extra worker:

```yaml
# kind-3workers.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

```powershell
kind create cluster --name k8s-lab-3 --config kind-3workers.yaml
kubectl apply -f log-agent-ds.yaml
kubectl get daemonset            # DESIRED should be 3
```

Expected behaviour: `DESIRED` equals the number of eligible nodes, and the DaemonSet controller creates or removes pods automatically as nodes join or leave.

---

## 10. Cleanup

```powershell
kubectl delete daemonset log-collector
kubectl label node kind-worker monitoring-            # only if the label is still set
kubectl taint nodes kind-worker2 dedicated=gpu:NoExecute-   # only if the taint is still set
kubectl uncordon kind-worker2                          # only if still cordoned
```

---

## 11. Conclusion

A DaemonSet deployed on a two-worker kind cluster scheduled **exactly one Fluent Bit pod per worker node** with no replica configuration. The experiment showed how placement is controlled: a **`nodeSelector`** limited the agent to labelled nodes, a **taint** excluded a node until a matching **toleration** was added, and a **cordon** did not stop the DaemonSet from keeping its pod on a node. These behaviours are why DaemonSets are the standard way to run logging, monitoring and networking agents across a cluster.

---


