

# **Kubernetes Component Controller Manager, CNI and kube-proxy**

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

