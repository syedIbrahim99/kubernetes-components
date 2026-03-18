## Kubernetes Pod Creation — Sequence Diagram

```bash
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

### Kubernetes Pod Creation Flow

1. You run
   `kubectl apply -f pod.yaml`

2. **kubectl → API Server**
   Sends request to create Pod

3. **API Server**

   * checks authentication & permissions
   * validates YAML

4. **API Server → etcd**
   Stores Pod details

5. **etcd → API Server → kubectl**
   Sends **200 OK**
   👉 Pod is created (but **not scheduled yet**)

---

6. **Scheduler watches API Server**
   Finds Pod with no node assigned

7. **Scheduler decides node**
   (based on CPU, RAM, etc.)

8. **Scheduler → API Server (bind request)**
   Assigns Pod to a node

9. **API Server → etcd**
   Updates Pod with node info

---

10. **kubelet (on that node) watches API Server**
    Detects Pod assigned to it

11. **kubelet → containerd**
    Tells to start container

12. **containerd**

* pulls image
* creates container
* starts container

---

13. **containerd → kubelet**
    Container started

14. **kubelet → API Server**
    Sends Pod status (Running)

15. **API Server → etcd**
    Stores updated status

---

## ✅ Final in one line (easy to remember)

👉
**kubectl → API Server → etcd → Scheduler → API Server → kubelet → containerd → kubelet → API Server → etcd**


