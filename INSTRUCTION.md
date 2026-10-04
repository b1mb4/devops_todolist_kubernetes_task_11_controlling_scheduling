# Instructions for Validating Kubernetes Scheduling Controls

This guide explains how to spin up the Kind cluster, apply node taints, deploy the application components, and validate the node affinity, pod anti-affinity, and tolerations configurations for both MySQL StatefulSet and ToDo App Deployment.

---

## 1. Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) (Kubernetes in Docker)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed and configured

---

## 2. Spin Up the Kind Cluster

Create the cluster using the provided `cluster.yml` configuration:

```bash
kind create cluster --config cluster.yml
```

Verify that all 7 nodes (1 control-plane and 6 workers) are running:

```bash
kubectl get nodes
```

---

## 3. Inspect Nodes for Labels and Taints

Check the labels assigned to the nodes:

```bash
kubectl get nodes --show-labels
```

Filter specifically to check `app` labels:

```bash
kubectl get nodes -L app
```

You should observe:
- 2 nodes with label `app=mysql`
- 3 nodes with label `app=todoapp`
- 1 worker node without an `app` label
- 1 control-plane node with `ingress-ready=true`

Check existing taints on all nodes:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

Initially, only the control-plane node has a taint (`node-role.kubernetes.io/control-plane:NoSchedule`).

---

## 4. Taint MySQL Nodes

Apply the `app=mysql:NoSchedule` taint to all nodes labeled with `app=mysql`:

```bash
kubectl taint nodes -l app=mysql app=mysql:NoSchedule --overwrite
```

Verify that the taint has been applied:

```bash
kubectl get nodes -l app=mysql -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

**Expected output:**
```
NAME                 TAINTS
cluster-worker       [map[effect:NoSchedule key:app value:mysql]]
cluster-worker2      [map[effect:NoSchedule key:app value:mysql]]
```

---

## 5. Deploy Resources via Bootstrap Script

Run the `bootstrap.sh` script to apply all manifests into the cluster:

```bash
chmod +x bootstrap.sh
./bootstrap.sh
```

Wait for all pods in both namespaces to be ready:

```bash
kubectl wait --for=condition=ready pod -l app=mysql -n mysql --timeout=180s
kubectl wait --for=condition=ready pod -l app=todoapp -n todoapp --timeout=180s
```

---

## 6. Validate StatefulSet (MySQL) Scheduling

### 6.1 Toleration of Tainted Nodes
Verify that MySQL pods are scheduled onto the worker nodes tainted with `app=mysql:NoSchedule`:

```bash
kubectl get pods -n mysql -o wide
```

Both `mysql-0` and `mysql-1` should be running on the tainted nodes (e.g., `cluster-worker` and `cluster-worker2`).

Inspect the tolerations in the StatefulSet manifest:

```bash
kubectl get statefulset mysql -n mysql -o jsonpath='{.spec.template.spec.tolerations}'
```

**Expected output contains:**
```json
[{"effect":"NoSchedule","key":"app","operator":"Equal","value":"mysql"}]
```

### 6.2 Node Affinity Rule (`app=mysql`)
Check the node affinity configuration:

```bash
kubectl get statefulset mysql -n mysql -o jsonpath='{.spec.template.spec.affinity.nodeAffinity}'
```

Verify that MySQL pods are strictly scheduled on nodes matching `app=mysql`:
- Pods are assigned to nodes that have the label `app=mysql`.

### 6.3 Pod Anti-Affinity Rule (Distinct Nodes)
Check the pod anti-affinity configuration:

```bash
kubectl get statefulset mysql -n mysql -o jsonpath='{.spec.template.spec.affinity.podAntiAffinity}'
```

Verify that `mysql-0` and `mysql-1` are on distinct nodes:

```bash
kubectl get pods -n mysql -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

The node names for `mysql-0` and `mysql-1` must be different.

---

## 7. Validate Deployment (ToDo App) Scheduling

### 7.1 Preferred Node Affinity Rule (`app=todoapp`)
Check the node affinity configuration:

```bash
kubectl get deployment todoapp -n todoapp -o jsonpath='{.spec.template.spec.affinity.nodeAffinity}'
```

**Expected output contains:**
```json
{"preferredDuringSchedulingIgnoredDuringExecution":[{"preference":{"matchExpressions":[{"key":"app","operator":"In","values":["todoapp"]}]},"weight":1}]}
```

Verify that the ToDo App pods are placed on nodes labeled `app=todoapp`:

```bash
kubectl get pods -n todoapp -o wide
```

### 7.2 Pod Anti-Affinity Rule (Distinct Nodes)
Check the pod anti-affinity configuration:

```bash
kubectl get deployment todoapp -n todoapp -o jsonpath='{.spec.template.spec.affinity.podAntiAffinity}'
```

**Expected output contains:**
```json
{"requiredDuringSchedulingIgnoredDuringExecution":[{"labelSelector":{"matchExpressions":[{"key":"app","operator":"In","values":["todoapp"]}]},"topologyKey":"kubernetes.io/hostname"}]}
```

Verify that ToDo App pods are placed on different nodes:

```bash
kubectl get pods -n todoapp -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

### 7.3 Testing Anti-Affinity Under Scaling
Scale the ToDo App deployment to 3 replicas:

```bash
kubectl scale deployment todoapp -n todoapp --replicas=3
kubectl rollout status deployment todoapp -n todoapp
```

Check the node distribution of the 3 pods:

```bash
kubectl get pods -n todoapp -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

All 3 replicas should be placed on 3 separate nodes labeled `app=todoapp`.

If you scale to 5 replicas (with only 4 untainted worker nodes available):

```bash
kubectl scale deployment todoapp -n todoapp --replicas=5
kubectl get pods -n todoapp
```

The 5th replica will remain in `Pending` state with reason `0/7 nodes are available: ... pod anti-affinity rules`, confirming that hard pod anti-affinity is strictly enforced.

---

## 8. Cleanup

To delete the test cluster when finished:

```bash
kind delete cluster
```
