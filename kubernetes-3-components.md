

# **Kubernetes Components**
- Controller Manager
- CNI 
- kube-proxy
- CoreDNS

## 1️⃣ **Controller Manager — The “Cluster Supervisor”**

The **Controller Manager** acts as the **cluster boss**, always making sure the cluster matches the desired state.

### **Flow**

1. **Controller Manager checks with the API Server**

   > "Hey API Server, is there any new Deployment, ReplicaSet, or Pod that needs attention?"

2. **API Server replies**

   > "Yes! There’s a new Deployment here."

3. **Controller Manager makes decisions**

   > "Okay, I need 3 replicas. I’ll create 3 Pods."

4. **Controller Manager instructs API Server**

   > "API Server, please create these 3 Pods."

5. **API Server stores this in etcd** → etcd replies **200 OK**

   > "Pods stored safely!"

6. **Controller Manager receives confirmation**

   > "Great, boss (API Server), my work is done. Pods are created and saved."

### **Key Points**

* Controller Manager **never touches nodes directly**.
* It ensures **desired state** is maintained by creating/updating objects via the API Server.
* Works continuously in the background.

---

## 2️⃣ **CNI (Container Network Interface) — The “Network Engineer”**

The **CNI plugin** acts as a **network engineer for Pods**, setting up networking so Pods can communicate inside the cluster and outside world.

### **Flow (Pod starting on Node2)**

1. **Kubelet detects Pod assigned to its node**

   > "Hey Network Engineer (CNI), I need a network for this new Pod. Can you set it up?"

2. **CNI plugin sets up network**

   * Assigns **Pod IP**
   * Creates **veth pair** (virtual ethernet connection)
   * Connects Pod to node’s network
   * Configures routing

3. **CNI sends acknowledgment to kubelet**

   > "All done! The Pod now has an IP and can talk to the network."

4. **Kubelet proceeds to start the container**

### **Key Points**

* Each kubelet talks **only to its node’s CNI**.
* **Pod cannot start** until CNI confirms networking is ready.
* CNI ensures Pods have **IP addresses and connectivity** within the cluster.

---

## 3️⃣ **kube-proxy — The “Traffic Controller”**

The **kube-proxy** acts as a **traffic police** for cluster services, making sure requests reach the right Pod.

### **Scenario: Browser → NodePort Service → Pod**

**Setup:**

* Pod running Nginx on Node2: `10.244.1.5`
* Service called `nginx-service` of type NodePort, exposed on port `30080`
* kube-proxy runs on all nodes (Node1, Node2, …)

### **Story-Like Flow**

1. **Browser sends request to NodePort**

   * URL: `http://<NodeIP>:30080`
   * Browser only knows NodeIP + NodePort, not Pod IP

2. **Request reaches kube-proxy on the node**

   * kube-proxy sees traffic for NodePort `30080`
   * Checks with API Server which Pods belong to `nginx-service`

3. **kube-proxy forwards request to Pod**

   * Example: `10.244.1.5:80`
   * If multiple Pods exist, it load balances (round-robin)

4. **Pod handles request and responds**

   * Nginx Pod sends back HTML response

5. **kube-proxy forwards response back to browser**

   * Browser receives the page, unaware of actual Pod IP

### **Visualization**

```
Browser
   |
   v
NodeIP:NodePort (30080)
   |
   v
kube-proxy (on that node)
   |
   v
Pod IP (10.244.1.5)
   |
   v
Response back to Browser
```

### **Key Points**

* **Browser → Service → kube-proxy → Pod**
* Browser never sees Pod IP.
* kube-proxy continuously updates **routing rules** for NodePort, ClusterIP, or LoadBalancer services.
* Handles **Pod changes dynamically** — new Pods added or removed are automatically included/excluded.

---

# 4️⃣ CoreDNS and Kube-proxy Explanation:

# 🌐 **Cluster Setup With Predefined IPs**

| Component   | Name      | IP         | Node | Notes                                |
| ----------- | --------- | ---------- | ---- | ------------------------------------ |
| Master Node | k8-master | 10.0.0.131 | -    | API Server, Controller Manager, etcd |
| Worker Node | worker-1  | 10.0.0.132 | -    | Runs Pods                            |
| Worker Node | worker-2  | 10.0.0.133 | -    | Runs Pods                            |
| Worker Node | worker-3  | 10.0.0.134 | -    | Runs Pods                            |

---

## **Pods**

| Pod               | Pod IP      | Node     |
| ----------------- | ----------- | -------- |
| nginx             | 10.244.1.10 | worker-1 |
| backend           | 10.244.3.20 | worker-3 |
| observer-frontend | 10.244.2.15 | worker-2 |
| builder-backend   | 10.244.2.25 | worker-2 |

---

## **Services (ClusterIP)**

| Service           | ClusterIP      | Port | Maps to Pods                      |
| ----------------- | -------------- | ---- | --------------------------------- |
| backend           | 10.110.147.141 | 2000 | backend Pod 10.244.3.20           |
| observer-frontend | 10.107.188.248 | 3000 | observer-frontend Pod 10.244.2.15 |
| builder-backend   | 10.110.147.142 | 7860 | builder-backend Pod 10.244.2.25   |

---

## **DNS / CoreDNS**

* CoreDNS Service IP: 10.96.0.10
* Converts **service name → ClusterIP**

Example:

```text
backend.autox-dev.svc.cluster.local → 10.110.147.141
```

---

# 🚀 **Step-by-Step Flow: nginx Pod → backend Service**

### **1️⃣ Application Sends Request**

Inside nginx Pod (worker-1, IP 10.244.1.10):

```bash
curl http://backend:2000
```

* nginx Pod wants backend
* Backend Service is `backend.autox-dev.svc.cluster.local`

---

### **2️⃣ DNS Lookup (CoreDNS)**

Pod sends DNS query → CoreDNS (10.96.0.10):

```text
"backend.autox-dev.svc.cluster.local ?"
```

CoreDNS responds:

```text
backend → 10.110.147.141 (ClusterIP)
```

---

### **3️⃣ Packet Sent to Service IP**

Now request goes:

```text
Source Pod → Destination Service
10.244.1.10 → 10.110.147.141:2000
```

---

### **4️⃣ kube-proxy Intercepts (on worker-1)**

kube-proxy on **worker-1** sees destination = Service IP → redirects to real Pod:

```text
10.110.147.141 → 10.244.3.20 (backend Pod on worker-3)
```

This is done using **iptables/IPVS rules (DNAT)**

---

### **5️⃣ CNI Routes Traffic Between Nodes**

* Source: 10.244.1.10 (worker-1)
* Destination: 10.244.3.20 (worker-3)

CNI (Flannel / Calico) encapsulates and sends traffic:

```text
worker-1 → worker-3
```

---

### **6️⃣ Traffic Arrives at Backend Pod**

Backend Pod receives:

```text
10.244.3.20 → port 2000
```

Processes request.

---

### **7️⃣ Response Goes Back**

Reverse path:

```text
backend Pod (10.244.3.20) → nginx Pod (10.244.1.10)
```

CNI handles routing back.

---

# 🔄 **Full Flow Diagram With Fixed IPs**

```text
nginx Pod (10.244.1.10, worker-1)
   |
   | curl backend
   v
CoreDNS (10.96.0.10)
   | → returns 10.110.147.141 (backend Service)
   v
kube-proxy (worker-1) ✅ intercepts
   | DNAT: 10.110.147.141 → 10.244.3.20
   v
CNI → routes packet worker-1 → worker-3
   v
backend Pod (10.244.3.20, worker-3)
   |
   v
Response → nginx Pod
```

---

# 🧠 **Key Notes**

1. **kube-proxy only works on the source node**

   * Here: nginx Pod is on worker-1 → kube-proxy on worker-1 redirects Service IP → Pod IP

2. **CNI handles real network delivery**

   * Cross-node: worker-1 → worker-3
   * Same node: direct bridge

3. **Direct Pod IP bypasses kube-proxy**

   * If nginx Pod did `curl 10.244.3.20:2000` → kube-proxy NOT used

4. **CoreDNS is only for name → IP translation**

---


