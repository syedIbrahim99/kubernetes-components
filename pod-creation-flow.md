
# Kubernetes Pod Creation — Sequence Diagram

```
kubectl        API Server          etcd           scheduler         kubelet          containerd
   |               |                 |                 |               |                  |
1  |---create Pod->|                 |                 |               |                  |
   |               |---write Pod---->|                 |               |                  |
2  |               |<------200-------|                 |               |                  |
3  |<------200-----|                 |                 |               |                  |
   |               |                 |                 |               |                  |
4  |               |                 |                 |               |                  |
   |               |------------------------------->scheduler          |                  |
5  |               |                 |               |                 |                  |
   |               |                 |               |                 |                  |
6  |               |<---------------bind Pod---------|                 |                  |
   |               |---write update->|               |                 |                  |
7  |               |<------200-------|               |                 |                  |
   |               |                 |               |                 |                  |
8  |               |<-----------------------------------------------  kubelet             |
   |               |              Pod assigned to node                                    |
   |               |                 |                                 |                  |
9  |               |                 |                                 |-----start------->|
   |               |                 |                                 |     container    |
   |               |                 |                                 |                  |
10 |               |                 |                                 |< ---container----|
   |               |                 |                                 |      started     |
   |               |                 |                                 |                  |
11 |               |<-------------Pod status update--------------------|                  |
   |               |---write status->|                                 |                  |
12 |               |<------200-------|                                 |                  |
```

---

# Step-by-Step Flow 

---

# Step 1 — kubectl sends request to API Server

Command:

```
kubectl apply -f pod.yaml
```

Flow:

```
kubectl → API Server
```

kubectl message:

> "API Server, please create this pod."

API Server receives the request.

Before doing anything it checks:

• Authentication
• Authorization (RBAC)
• YAML validity

If valid, it proceeds.

---

# Step 2 — API Server stores pod specification in etcd

Now API Server must store the **desired state**.

Flow:

```
API Server → etcd
(write pod specification)
```

API Server message:

> "etcd, store this new Pod configuration."

etcd writes the data into its database.

---

# Step 3 — etcd sends acknowledgement

After storing the data:

```
etcd → API Server
200 OK
```

Meaning:

> "Pod specification successfully stored."

Now the API Server knows the pod definition is saved.

---

# Step 4 — API Server replies to kubectl

API Server now acknowledges the client.

```
API Server → kubectl
200 OK
```

kubectl prints something like:

```
pod/nginx created
```

Important:

At this moment:

✔ Pod exists in **etcd database**
❌ Pod is **not scheduled yet**

---

# Step 5 — Scheduler detects new Pod

The scheduler is **continuously watching the API Server** for unscheduled pods.

It sees:

```
Pod created
Node = not assigned
```

Flow:

```
Scheduler → API Server
Request pod details
```

API Server sends pod information.

Scheduler now evaluates nodes:

Example:

```
Node1 CPU full
Node2 available
Node3 memory low
```

Scheduler decides:

```
Run pod on Node2
```

---

# Step 6 — Scheduler binds Pod to Node

Scheduler sends a binding request.

```
Scheduler → API Server
Bind Pod → Node2
```

API Server must store this decision.

Flow:

```
API Server → etcd
(update pod node assignment)
```

etcd stores the update.

---

# Step 7 — etcd acknowledges update

```
etcd → API Server
200 OK
```

Now the database state becomes:

```
Pod
Node = Node2
Status = Pending
```

---

# Step 8 — Kubelet detects assigned pod

Every worker node runs **kubelet**.

Kubelet continuously watches API Server for pods assigned to its node.

Kubelet on **Node2** sees:

```
Pod assigned to Node2
```

Flow:

```
kubelet → API Server
Fetch pod specification
```

API Server returns pod definition.

Kubelet says:

> "Okay, I will start this pod."

---

# Step 9 — Kubelet asks container runtime to start container

Kubelet talks to the container runtime.

Example runtime:

```
containerd
```

Flow:

```
kubelet → containerd
Start container
```

containerd performs:

1️⃣ Pull image
2️⃣ Create container
3️⃣ Start container

---

# Step 10 — Container runtime confirms container start

After container starts:

```
containerd → kubelet
Container Started
```

Now the container is actually running.

---

# Step 11 — Kubelet sends Pod status update

Kubelet informs API Server.

Flow:

```
kubelet → API Server
Pod Status = Running
```

API Server must store the updated status.

Flow:

```
API Server → etcd
(update pod status)
```

---

# Step 12 — etcd sends final acknowledgement

```
etcd → API Server
200 OK
```

Now etcd stores:

```
Pod Status = Running
Node = Node2
```

Cluster state is now updated.

---

# Final Result

```
kubectl
   |
   v
API Server
   |
   v
etcd  (store pod spec)
   |
   v
scheduler (choose node)
   |
   v
API Server
   |
   v
etcd (store node assignment)
   |
   v
kubelet (worker node)
   |
   v
containerd (start container)
   |
   v
Pod Running
```

---

# Summary 

```
kubectl sends request
API Server stores in etcd
scheduler assigns node
kubelet detects assignment
container runtime starts container
kubelet updates pod status
API Server stores status in etcd
```

---


