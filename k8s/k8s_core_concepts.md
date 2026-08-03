# Kubernetes (K8s) Architecture & Core Components

# ☸️ Kubernetes (K8s) Architecture & Interview Guide

Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. It follows a **Control Plane (Master)** and **Worker Node** architecture.

+-------------------------------------------------+
                   |                  CONTROL PLANE                  |
                   |                                                 |
                   |   +-------------------+   +-----------------+   |
                   |   |   kube-apiserver  |   |      etcd       |   |
                   |   +-------------------+   +-----------------+   |
                   |             |                      |            |
                   |   +-------------------+   +-----------------+   |
                   |   |  kube-scheduler   |   | controller-mgr  |   |
                   |   +-------------------+   +-----------------+   |
                   +-------------------------------------------------+
                                            |
                           +----------------+----------------+
                           |                                 |
                           v                                 v
           +-------------------------------+ +-------------------------------+
           |          WORKER NODE          | |          WORKER NODE          |
           |                               | |                               |
           |  +---------+   +-----------+  | |  +---------+   +-----------+  |
           |  | kubelet |   |kube-proxy |  | |  | kubelet |   |kube-proxy |  |
           |  +---------+   +-----------+  | |  +---------+   +-----------+  |
           |  +-------------------------+  | |  +-------------------------+  |
           |  |    Container Runtime    |  | |  |    Container Runtime    |  |
           |  +-------------------------+  | |  +-------------------------+  |
           |  +-------------------------+  | |  +-------------------------+  |
           |  |     Pods (Containers)   |  | |  |     Pods (Containers)   |  |
           |  +-------------------------+  | |  +-------------------------+  |
           +-------------------------------+ +-------------------------------+


## 🧠 1. Control Plane Architecture (The Brain)

The Control Plane maintains the desired state of the cluster, makes global scheduling decisions, and handles cluster events.

### 🚪 `kube-apiserver`
* **What it is:** The primary REST API endpoint for the cluster.
* **How it works:** All communication—whether from external users using `kubectl`, worker nodes via `kubelet`, or internal controllers—must pass through the API Server. It authenticates, authorizes, and validates every request before saving data to `etcd`.

### 💾 `etcd`
* **What it is:** A distributed, consistent key-value store.
* **How it works:** It stores the entire state and configuration of the Kubernetes cluster (e.g., how many Pods should run, node health, secrets). It is the single source of truth for the cluster.

### 🗓️ `kube-scheduler`
* **What it is:** The resource assignment engine.
* **How it works:** When a new Pod is created, the scheduler looks for a suitable worker node to run it. It considers resource requirements (CPU/Memory), taints, tolerations, node affinity, and policy constraints.

### ⚙️ `kube-controller-manager`
* **What it is:** The continuous reconciliation loop engine.
* **How it works:** It runs several background controllers (Node Controller, ReplicaSet Controller, EndpointSlice Controller). Each controller constantly compares the **current state** of the cluster with the **desired state** (stored in `etcd`) and takes corrective action if there is a mismatch.

### ☁️ `cloud-controller-manager`
* **What it is:** The cloud integration bridge.
* **How it works:** Connects Kubernetes to cloud provider APIs (like AWS, Azure, or GCP) to automatically provision resources like cloud load balancers, storage volumes, and virtual network routes.

---

## 🚜 2. Worker Node Architecture (The Muscle)

Worker Nodes execute the actual containerized workloads assigned to them by the Control Plane.

### 🧑‍✈️ `kubelet`
* **What it is:** The node manager agent.
* **How it works:** It runs directly on every worker node. It receives instructions (`PodSpecs`) from the `kube-apiserver` and talks to the local Container Runtime to start, monitor, and stop containers.

### 🔀 `kube-proxy`
* **What it is:** The network routing agent.
* **How it works:** Runs on each node to manage network rules (using `iptables` or `IPVS`). It allows network traffic to reach Pods across nodes and load-balances traffic across Pods behind a Kubernetes Service.

### 📦 `Container Runtime`
* **What it is:** The engine that pulls and runs container images.
* **How it works:** Executes container operations adhering to the Container Runtime Interface (CRI). Examples include `containerd` and `CRI-O`.

---

## 🧩 3. Key Concepts Quick Reference

* 🥜 **Pod:** The smallest execution unit in K8s; encapsulates one or more containers sharing network IP and storage.
* 🌐 **Service:** An abstraction providing a stable IP address and DNS name for a set of dynamic Pods.
* 📈 **Deployment:** Manages stateless Pod replicas, enabling zero-downtime rolling updates and rollbacks.
* 📁 **Namespace:** Virtual partitions inside a physical cluster to isolate resources across teams or environments.

---

## 🎯 4. Essential Interview Questions & Answers

### Q1: What happens under the hood when you run `kubectl apply -f deployment.yaml`?
**Answer:**
1. **Authentication & Authorization:** `kubectl` sends an HTTP POST request to `kube-apiserver`. The API server verifies your credentials and RBAC permissions.
2. **Validation & Storage:** The API server validates the YAML structure and writes the new Deployment manifest into `etcd`.
3. **Deployment Controller:** The `kube-controller-manager` detects the new Deployment in `etcd` and creates a **ReplicaSet**.
4. **ReplicaSet Controller:** The ReplicaSet controller sees the required replica count and creates individual **Pod** objects in `etcd` with a status of `Pending`.
5. **Scheduling:** The `kube-scheduler` notices unscheduled Pods, evaluates available nodes, selects the best node, and updates the Pod object with the assigned node name.
6. **Execution:** The `kubelet` on the target node sees a Pod assigned to it via `kube-apiserver`. It commands the local **Container Runtime** to pull the image and launch the container.
7. **Networking:** `kube-proxy` configures networking rules so the Pod can communicate.

---

### Q2: Why is `etcd` deployed with an odd number of instances (3, 5, or 7)?
**Answer:**
`etcd` relies on the **Raft Consensus Algorithm** to maintain data consistency across a distributed system. 
* To reach a consensus, a majority (quorum) of nodes must agree: $\text{Quorum} = \lfloor N/2 \rfloor + 1$.
* A 3-node cluster can tolerate **1** failure ($3 - 2 = 1$). A 4-node cluster can also tolerate only **1** failure ($4 - 3 = 1$), but requires more overhead.
* Odd numbers optimize fault tolerance without wasting compute resources on unnecessary consensus nodes.

---

### Q3: What is the difference between `kubelet` and `kube-proxy`?
**Answer:**
* **`kubelet`** manages the **lifecycle of Pods** on a specific node (pulling images, starting/stopping containers, reporting node health).
* **`kube-proxy`** manages **network traffic and routing** on the node (updating routing tables, managing load balancing for Services across Pods).

---

### Q4: What is the difference between a Pod and a Container?
**Answer:**
* A **Container** is a single isolated process running an application with its own filesystem and dependencies.
* A **Pod** is a Kubernetes-specific abstraction that wraps around one or more closely related containers. Containers within the same Pod share the same **IP address**, **network port space**, and **storage volumes**, and can communicate via `localhost`.

---

### Q5: How does Kubernetes handle a Worker Node failure?
**Answer:**
1. The `kubelet` periodically sends heartbeats to `kube-apiserver`.
2. If a node stops sending heartbeats, the Node Controller waits for a grace period (default: `node-monitor-grace-period` = 40 seconds).
3. If the node remains unreachable, the controller marks the node status as `NotReady`.
4. After another timeout (`pod-eviction-timeout` = 5 minutes by default), the Control Plane schedules replacement Pods onto healthy worker nodes to restore the desired state.

ETCD: 
# 💾 `etcd` Deep Dive: Architecture & Mechanics

`etcd` is a strongly consistent, distributed key-value store that acts as the primary state store for Kubernetes. It is built to reliably hold data across a cluster of machines.

---

## 🏗️ 1. Core Architectural Components

`etcd` combines a consensus engine, an in-memory index, and a disk-backed store to manage data safely and efficiently:

* 🤝 **Raft Consensus Engine:** Guarantees that all healthy nodes agree on the exact same sequence of data modifications.
* ⚡ **gRPC API:** Uses HTTP/2 and Protocol Buffers to offer high-performance communication between clients (like `kube-apiserver`) and the cluster.
* 🌳 **In-Memory B-Tree Index:** Maps human-readable key names (e.g., `/registry/pods/default/nginx`) to specific revision numbers.
* 🗄️ **bbolt Key-Value Store:** An on-disk B+tree database that maps revision numbers to actual values and metadata.

---

## ⚙️ 2. How `etcd` Works Under the Hood

### Step A: Leader Election & Cluster Quorum 👑
1. On startup, nodes in the `etcd` cluster communicate using the **Raft protocol**.
2. A single node is elected as the **Leader**, while all other nodes become **Followers**.
3. All write requests must go through the Leader. If a write lands on a Follower, it is forwarded to the Leader.
4. To accept a write, the Leader must achieve a **Quorum** (agreement from a majority of nodes: $\lfloor N/2 \rfloor + 1$).

---

### Step B: The Lifecycle of a Write Operation ✍️

When `kube-apiserver` writes a new resource into `etcd`:

1. **Log Append:** The Leader receives the request and appends the new entry to its own local Raft log.
2. **Replication:** The Leader sends an `AppendEntries` RPC to all Follower nodes.
3. **Quorum Acknowledgment:** Once a majority of Followers write the entry to their logs, they respond with success.
4. **Commit & Apply:** The Leader commits the entry, applies the change to its state store, updates its in-memory index, and returns success to the client.
5. **Follower Apply:** The Leader notifies Followers on the next heartbeat, and Followers apply the committed entry to their state stores.

---

### Step C: Multi-Version Concurrency Control (MVCC) 📜
* `etcd` **never overwrites data in place**. Instead, every write creates a new **Revision** (a globally incrementing counter).
* Old versions of keys remain stored until a **Compaction** process cleans them up.
* This version history allows Kubernetes to implement optimistic locking (preventing race conditions) and safely roll back state.

---

### Step D: Real-Time Updates via the Watch API 👁️
* Rather than requiring clients to continuously poll for changes, `etcd` uses a long-lived **gRPC Watch Stream**.
* When a key or key-prefix changes, `etcd` immediately pushes the new revision event to all subscribed watchers (e.g., Kubernetes controllers).

kubeAPI server: 

The **`kube-apiserver`** is the central management engine of the Kubernetes Control Plane. It serves as the primary RESTful API gateway through which all administrative actions, internal operations, and worker node communications flow.

---

## 1. Core Architecture & High-Level Responsibilities

```text
                                 [ HTTP / HTTPS Request ]
                                            |
                                            v
                      +-------------------------------------------+
                      |              kube-apiserver               |
                      |                                           |
                      |  1. Authentication   (Tokens/Certs)      |
                      |  2. Authorization      (RBAC/ABAC)        |
                      |  3. Admission Control  (Mutating/Validating)|
                      |  4. Schema Validation  (OpenAPI Specs)    |
                      +-------------------------------------------+
                                            |
                                            v
                                   +-----------------+
                                   |      etcd       |
                                   |  (Key-Value DB) |
                                   +-----------------+


Key Functions
Central API Gateway: Exposes the REST API endpoint (typically on HTTPS TCP port 6443) for all CLI clients (kubectl, Helm), automated systems, and worker nodes.

Exclusive Gatekeeper to etcd: The kube-apiserver is the only component in the entire cluster that directly reads from or writes to etcd.

Stateless Processing: Holds no persistent state locally. It validates, mutates, authorizes, and serializes requests in memory before storing them in etcd.

Declarative State Validation: Ensures all submitted YAML/JSON manifests comply with Kubernetes OpenAPI schema rules.

The 4-Stage Request Processing Lifecycle
When an API request (such as kubectl apply -f pod.yaml) reaches the kube-apiserver, it passes through four distinct execution stages before the state is written to etcd.

[ Incoming Request ] ──> [ 1. Authentication ] ──> [ 2. Authorization (RBAC) ] ──> [ 3. Admission Control ] ──> [ 4. etcd Write ]

----------------------------------------------------
Stage 1: Authentication (Who are you?)
Verifies the client's identity using various mechanisms:

X.509 Client Certificates (e.g., administrator keys)

Bearer Tokens & ServiceAccount Tokens (used by pods and automated workflows)

OpenID Connect (OIDC) (integration with Okta, Google Workspace, Azure AD)

Stage 2: Authorization (Are you allowed to do this?)
Evaluates if the authenticated identity has permission to perform the requested verb (get, list, create, update, delete) on the target resource:

RBAC (Role-Based Access Control): The industry standard mechanism using Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings.

ABAC (Attribute-Based Access Control): Policy-based access using arbitrary attributes.

Stage 3: Admission Control (Should this request be allowed or altered?)
Processes the request through a series of admission controller plugins:

Mutating Admission Webhooks: Modifies or injects fields into the incoming object (e.g., injecting an Istio sidecar container or assigning default storage classes).

Schema Validation: Verifies structural compliance against OpenAPI specifications.

Validating Admission Webhooks: Inspects the final object state against policy engines (e.g., Kyverno, OPA Gatekeeper). If validation fails, the request is rejected immediately.

Stage 4: Persistence to etcd
Once all checks pass, kube-apiserver serializes the API object and stores it inside etcd.

4. High Availability (HA) & Scaling
Active-Active Stateless Design: Because kube-apiserver does not store local data, multiple instances run concurrently behind a Layer-4 or Layer-7 Load Balancer.

Direct etcd Synchronization: All API server instances talk to the underlying etcd cluster. Leader election is not required for kube-apiserver instances (unlike kube-scheduler or kube-controller-manager).

----------------------------------------------------------

Top Interview Questions
Q1: Why is kube-apiserver the ONLY component that talks directly to etcd?
Answer: Restricting etcd access exclusively to the API server ensures strict data consistency, prevents race conditions, enforces central authentication/authorization, and validates schemas before persistence.

Q2: What is the difference between Mutating and Validating Admission Webhooks?
Answer: Mutating webhooks run first and can modify or inject fields into the object (e.g., sidecars). Validating webhooks run second and inspect the finalized object; they cannot modify objects, only accept or reject requests.

Q3: What happens to running applications if all kube-apiserver instances go offline?
Answer: Existing containers on worker nodes continue running uninterrupted because kubelet and kube-proxy use cached rules. However, administrative commands (kubectl), pod reschedules, auto-scaling, and new deployments will fail until the API server recovers.

Q4: How does kubectl get pods -w (watch mode) work without overloading etcd?
Answer: kube-apiserver uses an HTTP Long-Polling / Server-Sent Events (SSE) watch stream. It pushes real-time delta updates directly from its internal memory cache to the subscriber without repeatedly querying etcd.

Q5: How do you scale kube-apiserver for High Availability (HA)?
Answer: Since kube-apiserver is stateless, multiple active instances run concurrently behind a Layer-4 or Layer-7 Load Balancer, all reading and writing to the shared etcd cluster.


Admisssion webhook : 

An Admission Webhook is an HTTP callback mechanism inside Kubernetes that intercepts API requests to the kube-apiserver after they are authenticated and authorized, but before the object state is stored in etcd.
Admission webhooks allow developers and cluster operators to enforce custom business logic, security policies, and automated modifications to Kubernetes resources without changing the core Kubernetes codebase.

1. Where Do Admission Webhooks Sit in the Request Lifecycle?
When you execute a command like kubectl apply -f deployment.yaml, the request moves through four main phases inside the kube-apiserver:

[ API Request ]
                                       |
                                       v
                             [ 1. Authentication ]
                                       |
                                       v
                           [ 2. Authorization (RBAC) ]
                                       |
                                       v
  +--------------------------------------------------------------------------+
  |                        3. ADMISSION CONTROL                              |
  |                                                                          |
  |  +-----------------------------------+                                   |
  |  | Mutating Admission Webhooks       |  <-- Modifies or injects fields   |
  |  +-----------------------------------+                                   |
  |                    |                                                     |
  |                    v                                                     |
  |  +-----------------------------------+                                   |
  |  | Schema Validation                 |  <-- Checks OpenAPI syntax        |
  |  +-----------------------------------+                                   |
  |                    |                                                     |
  |                    v                                                     |
  |  +-----------------------------------+                                   |
  |  | Validating Admission Webhooks     |  <-- Accepts or rejects object    |
  |  +-----------------------------------+                                   |
  +--------------------------------------------------------------------------+
                                       |
                                       v
                               [ 4. etcd Write ]


2. The Two Types of Admission Webhooks
Kubernetes supports two distinct types of admission webhooks, which execute sequentially:

A. Mutating Admission Webhooks
When they run: First.

What they do: They can modify or alter the incoming resource object before it gets validated and saved.

Common Use Cases:

Sidecar Injection: Automatically injecting an Istio or Linkerd service mesh proxy container into every application Pod.

Default Values: Adding default resource requests/limits (CPU and Memory) if the user didn't specify them.

Storage Configuration: Adding custom labels, annotations, or security contexts automatically.

B. Validating Admission Webhooks
When they run: Second (after mutating webhooks and schema validation finish).

What they do: They inspect the finalized object to accept or reject the request. They cannot modify the object.

Common Use Cases:

Security Enforcement: Blocking any Pod that attempts to run as the root user (runAsNonRoot: true).

Naming/Label Enforcement: Rejecting deployments that lack mandatory environment or ownership tags (e.g., team: dev).

Image Registry Restriction: Preventing developers from pulling container images from unauthorized public registries (e.g., forcing images to come only from mycompany.azurecr.io).


# Comprehensive Guide to Kyverno & Kubernetes Admission Webhooks

**Kyverno** (Greek for "govern") is an open-source, Kubernetes-native policy engine designed specifically for Kubernetes. Unlike general-purpose policy engines (such as Open Policy Agent / OPA Gatekeeper, which require learning a separate programming language like Rego), Kyverno allows you to write policies as **standard Kubernetes CRDs (Custom Resource Definitions) in YAML**.

---

## 1. How Kyverno is Built Around Admission Webhooks

Kyverno runs inside your Kubernetes cluster as a controller deployment alongside an **Admission Controller** service. It leverages Kubernetes Dynamic Admission Control to enforce, mutate, validate, and generate resources.

```text
                                [ User / kubectl / CI/CD ]
                                            |
                                            v
                                  [ kube-apiserver ]
                                            |
                             (HTTP AdmissionReview Payload)
                                            |
                                            v
     +-------------------------------------------------------------------------------+
     |                            KYVERNO CONTROLLER                                 |
     |                                                                               |
     |   +-----------------------------------------------------------------------+   |
     |   |                   Mutating Webhook Endpoint                           |   |
     |   |   (Applies default values, annotations, or sidecar injections)        |   |
     |   +-----------------------------------------------------------------------+   |
     |                                      |                                        |
     |                                      v                                        |
     |   +-----------------------------------------------------------------------+   |
     |   |                   Validating Webhook Endpoint                         |   |
     |   |   (Checks policies: returns ALLOW or DENY + error message)            |   |
     |   +-----------------------------------------------------------------------+   |
     |                                      |                                        |
     |                                      v                                        |
     |   +-----------------------------------------------------------------------+   |
     |   |                  Background Controller / Engine                       |   |
     |   |   (Scans existing resources & GENERATES new resources if needed)      |   |
     |   +-----------------------------------------------------------------------+   |
     +-------------------------------------------------------------------------------+
                                            |
                                            v
                                     [ etcd Write ]


The Architecture Workflow
Webhook Registration: When Kyverno is installed via Helm or YAML manifests, it automatically registers a MutatingWebhookConfiguration and a ValidatingWebhookConfiguration with the kube-apiserver.

Interception: When an API request (e.g., creating a Pod or Deployment) hits the kube-apiserver, the API server sends an AdmissionReview JSON object to Kyverno's webhook endpoints over HTTPS.

Policy Matching: Kyverno inspects its internal memory cache of active ClusterPolicy or Policy rules to see if any match the incoming resource (by Kind, Namespace, Labels, User Roles, etc.).

Execution:

Mutating Webhook Phase: Kyverno modifies the request payload if matching mutation rules exist (e.g., injecting security parameters).

Validating Webhook Phase: Kyverno evaluates validation rules. If a rule fails and the validationFailureAction is set to Enforce, Kyverno returns a DENY response, causing kube-apiserver to block the operation.

Background Scans & Policy Reports: In addition to webhooks, Kyverno runs a background controller that audits existing cluster resources against active policies and generates PolicyReport CRDs so you can see non-compliant resources without breaking them.                                     


kube-controller-manager:


The **`kube-controller-manager`** is a fundamental component of the Kubernetes Control Plane. It runs continuously as a single daemon binary that encapsulates multiple core control loops (controllers) responsible for regulating the state of the cluster.

---

## 1. Core Architecture & Concept: The Reconciliation Loop

In Kubernetes, you specify the **Desired State** of your cluster using declarative YAML manifests (e.g., "I want 3 replicas of my web app"). The `kube-controller-manager` is responsible for making sure the **Actual State** matches that **Desired State**.

It operates via a continuous loop known as the **Reconciliation Loop**:

```text
               +-------------------------------------------+
               |            RECONCILIATION LOOP            |
               |                                           |
               |  1. Observe Actual State (from etcd)      |
               |                     |                     |
               |                     v                     |
               |  2. Compare Actual vs. Desired State      |
               |                     |                     |
               |                     v                     |
               |  3. Take Action to Reconcile Differences  |
               +-------------------------------------------+
                                     ^
                                     |
                                     +--- (Loop continuously)

 Key Architectural Characteristics
Control Loop Abstraction: Rather than running dozens of separate background processes, Kubernetes packages multiple built-in controllers into a single process (kube-controller-manager) to reduce resource overhead and simplify operational management.

Stateless Operation: Like kube-scheduler and kube-apiserver, the controller manager does not hold persistent state locally. It queries the kube-apiserver (which reads from etcd) to inspect resource states and submit API calls for any required changes.

2. Key Built-in Controllers Inside kube-controller-manager
While bundled in one binary, several distinct logical controllers execute inside kube-controller-manager. Key controllers relevant for DevOps and Platform Engineering include:

A. Workload & Deployment Controllers
Deployment Controller: Listens for changes to Deployment objects. It manages the creation, rolling update, and rollback of underlying ReplicaSet objects.

ReplicaSet Controller: Ensures that the exact number of identical Pods specified in a ReplicaSet manifest are running at any given time. If a Pod crashes, it triggers the creation of a replacement.

StatefulSet Controller: Manages ordered startup, teardown, and persistent storage attachment for stateful applications.

DaemonSet Controller: Ensures that a copy of a designated Pod runs on every eligible worker node (e.g., node monitoring agents, log collectors).

Job / CronJob Controller: Spawns Pods to execute finite tasks and cleans them up upon completion or according to schedule.

B. Infrastructure & Cluster Operations Controllers
Node Controller:

Monitors node health by receiving heartbeat signals from each node's kubelet.

If a node stops sending heartbeats (exceeds node-monitor-grace-period, default ~40s), the Node Controller marks the node as NotReady or Unreachable.

If the node remains unresponsive (exceeds pod-eviction-timeout, default ~5m), it initiates pod eviction to reschedule workloads onto healthy nodes.

Endpoints / EndpointSlice Controller: Watches Service and Pod resources to populate Endpoints or EndpointSlice objects, mapping abstract network services to healthy pod IP addresses.

Namespace Controller: Cleans up all objects inside a deleted Namespace and removes the namespace once empty.

ServiceAccount Controller: Automatically creates default ServiceAccounts and API access tokens for new namespaces.

3. High Availability (HA) & Leader Election
Unlike kube-apiserver (which runs active-active), kube-controller-manager operates in an Active-Passive (Leader-Follower) model in high-availability clusters:

Plaintext
    +-----------------------------------------------------------------------+
    |                         LOAD BALANCER / API SERVER                    |
    +-----------------------------------------------------------------------+
            |                                   |
            v                                   v
+-----------------------+           +-----------------------+
|  Controller-Manager   |           |  Controller-Manager   |
|      (Instance 1)     |           |      (Instance 2)     |
|      [ LEADER ]       |           |     [ FOLLOWER ]      |
|                       |           |                       |
| Active Control Loops  |           | Idle / Waiting for    |
| Reconciling Cluster   |           | Lease Expiration      |
+-----------------------+           +-----------------------+
Why Leader Election? Having two active controllers trying to create or delete the same Pod or ReplicaSet simultaneously would cause race conditions and infinite creation loops.

How it Works: All controller manager instances attempt to acquire a distributed lock (a Lease object in the kube-system namespace) via the kube-apiserver. The instance that secures the lease becomes the Leader and performs all reconciliation loops. Follower instances remain idle, continuously renewing their attempt to acquire the lease if the primary leader fails.


Platform Engineering & DevOps Interview Questions & Answers :

Q1: What is the primary role of the kube-controller-manager in Kubernetes?
Answer: The kube-controller-manager is the core control loop engine of the Kubernetes Control Plane. It continuously monitors the current state of cluster resources via the kube-apiserver, compares it against the declared desired state, and executes actions to align the actual state with the desired state (e.g., creating missing pods, handling rolling updates, or evicting pods from dead nodes).

Q2: How does kube-controller-manager achieve High Availability (HA), and why does it use a Leader-Follower model instead of Active-Active?
Answer: It uses a Leader-Follower (Active-Passive) model managed via Kubernetes Lease API locks.
It cannot run Active-Active because reconciliation loops must be deterministic. If multiple controller instances were actively modifying resources at the same time, they would enter race conditions—for instance, both detecting a missing Pod and each launching a new duplicate Pod. By using a leader election mechanism (--leader-elect=true), only one active instance reconciles resources while secondary instances standby to take over if the leader drops its heartbeat lease.

Q3: What happens behind the scenes when a Worker Node suddenly dies or loses network connectivity?
Answer:

Heartbeat Loss: The kubelet on the failed node stops sending heartbeats to the kube-apiserver.

Node Controller Detection: The Node Controller inside kube-controller-manager notices the missing heartbeats. Once node-monitor-grace-period (default: 40s) elapses, it marks the node status as Unknown / NotReady.

Taint Application: The Node Controller automatically applies node.kubernetes.io/unreachable taints to the node.

Eviction Execution: Once pod-eviction-timeout (default: 5 minutes) passes, the Node Controller marks the pods on the dead node for deletion.

Rescheduling: The workload's Deployment/ReplicaSet controller notices the drop in running pod count and creates new Pod requests, which the kube-scheduler places onto healthy worker nodes.

Q4: What is the difference between a Built-in Controller in kube-controller-manager and a Custom Controller / Operator (e.g., using Kubebuilder / Operator SDK)?
Answer:

Built-in Controllers: Native, compiled C++/Go control loops compiled into the kube-controller-manager binary that manage core Kubernetes API resources (Pods, Deployments, Services, Namespaces).

Custom Controllers / Operators: External control loops deployed as standard workloads in the cluster. They manage Custom Resource Definitions (CRDs) to extend Kubernetes capabilities for complex domain-specific applications (e.g., Prometheus Operator, Cert-Manager, Postgres Operator). They follow the exact same reconciliation loop logic as native controllers.

Q5: How do Deployment controllers handle Rolling Updates without downtime?
Answer:
When a Deployment spec changes (e.g., updating a container image):

The Deployment Controller creates a new ReplicaSet alongside the existing one.

It scales up the new ReplicaSet incrementing by maxSurge (e.g., adding 1 new Pod).

The Endpoints Controller adds the newly created, healthy Pod (passing readiness probes) to the Service Endpoints.

The Deployment Controller then scales down the old ReplicaSet decrementing by maxUnavailable.

This step-by-step cycle repeats until the new ReplicaSet reaches 100% desired capacity and the old ReplicaSet scales down to 0.

Q6: If a Pod managed by a Deployment crashes, which component is directly responsible for recreating it—kubelet, kube-scheduler, or kube-controller-manager?
Answer:

kube-controller-manager (ReplicaSet Controller): Detects that the actual number of running Pods is lower than the desired count in the manifest and creates a new Pod object in etcd via kube-apiserver.

kube-scheduler: Detects the newly created Pod object (which has no assigned node) and binds it to an optimal worker node.

kubelet: Detects that a Pod has been assigned to its local node, communicates with the local container runtime to pull the image, and starts the container.

------------------------------------------------------------------------------------------------
# Understanding `maxSurge` and `maxUnavailable` in Kubernetes

When updating a Kubernetes **Deployment** (such as updating a container image version), Kubernetes performs a **Rolling Update** by default. To prevent application downtime during these updates, Kubernetes uses two key strategy fields: **`maxSurge`** and **`maxUnavailable`**.

These two fields control **how fast** an update proceeds and **how much capacity buffer/downtime** is allowed during the update process.

---

## 1. Quick Definitions

| Parameter | What it Controls | Default Value |
| :--- | :--- | :--- |
| **`maxSurge`** | The maximum number of **extra Pods** that can be created *above* the desired replica count during an update. | `25%` |
| **`maxUnavailable`** | The maximum number of **Pods that can be offline/unavailable** relative to the desired replica count during an update. | `25%` |

> 💡 **Key Rule:** Both parameters can be specified either as an **absolute number** (e.g., `2`) or as a **percentage** of the desired replicas (e.g., `25%`).

---

## 2. Deep Dive & Examples

Assume you have a Deployment with **`replicas: 4`**.

### A. `maxSurge` (The Upper Bound / Extra Capacity)
`maxSurge` defines how many additional Pods Kubernetes can launch temporarily while upgrading.

* **Example with Absolute Number (`maxSurge: 2`):**
  * Desired Replicas: `4`
  * Maximum Pods allowed during rollout: `4 + 2 = 6`
  * **Behavior:** Kubernetes can immediately spin up 2 new version Pods. Once those pass readiness probes, it starts terminating old Pods.

* **Example with Percentage (`maxSurge: 25%`):**
  * $25\% \text{ of } 4 = 1$
  * Maximum Pods allowed during rollout: `4 + 1 = 5`

---

### B. `maxUnavailable` (The Lower Bound / Capacity Drop)
`maxUnavailable` defines how many existing Pods can be destroyed before new replacement Pods are ready and operational.

* **Example with Absolute Number (`maxUnavailable: 1`):**
  * Desired Replicas: `4`
  * Minimum healthy Pods required during rollout: `4 - 1 = 3`
  * **Behavior:** Kubernetes can terminate 1 old Pod immediately, dropping running capacity to 3 while it spins up a new Pod to replace it.

* **Example with Percentage (`maxUnavailable: 25%`):**
  * $25\% \text{ of } 4 = 1$
  * Minimum healthy Pods required: `4 - 1 = 3`

---

## 3. How They Work Together: Common Configuration Patterns

How you combine `maxSurge` and `maxUnavailable` in your Deployment manifest depends entirely on your application's resource constraints and availability targets.

### Configuration YAML Spec Location
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  template:
    ...

# Best Rolling Update Strategy for Production

For production environments, the recommended standard configuration is **Zero-Downtime Rolling Updates**:

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0

1. Why maxSurge: 25% and maxUnavailable: 0 is the Gold Standard
A. Zero Capacity Drop (maxUnavailable: 0)
Guarantees that 100% of your serving capacity remains available at all times during a rollout.

Existing pods are never terminated until new replacement pods are completely created, initialized, and pass their Readiness Probes.

Prevents traffic overload on remaining pods during peak production load.

B. Controlled Burst (maxSurge: 25% or maxSurge: 1)
Provisions extra pods in small, controlled batches (e.g., 1 extra pod for a 4-replica setup).

Prevents sudden CPU/Memory spikes on cluster worker nodes while still keeping rollout times fast.

-------------------------------------------------
kube-scheduler: 


The **`kube-scheduler`** is a core component of the Kubernetes Control Plane. Its primary responsibility is **Pod Placement**: watching for newly created Pods that have no assigned worker node and assigning them to the most optimal node in the cluster based on available resources, constraints, and affinity rules.

---

## 1. Role & High-Level Architecture

```text
  +-----------------------+
  |  New Pod Created      |  (nodeName: "")
  |  and written to etcd  |
  +-----------------------+
              |
              v
  +---------------------------------------------------------------------------------+
  |                                 KUBE-SCHEDULER                                  |
  |                                                                                 |
  |   1. Filtering (Predicates)  ──>  2. Scoring (Priorities)  ──>  3. Binding      |
  |   (Find Feasible Nodes)           (Rank Eligible Nodes)         (Assign Node)   |
  +---------------------------------------------------------------------------------+
                                          |
                                          v
                              +-----------------------+
                              |  Update kube-apiserver|  (nodeName: "node-01")
                              |  (Persist to etcd)    |
                              +-----------------------+

Key Responsibilities
Monitors Unschedulable Pods: Continuously watches kube-apiserver for Pods where spec.nodeName is empty.

Optimal Placement: Finds the best fit for a Pod rather than assigning it to a random node.

Does NOT Execute Pods: kube-scheduler only assigns the node name (spec.nodeName) in the Pod manifest. It is the worker node's kubelet agent that detects this binding and actually runs the containers.

2. The 2-Phase Scheduling Process: Filtering & Scoring
When scheduling a Pod, kube-scheduler evaluates all worker nodes in the cluster through a two-phase pipeline:

Phase 1: Filtering (Predicates)
Filters out nodes that cannot run the Pod due to physical or logical constraints. If no nodes pass this phase, the Pod remains in a Pending state.

Common Filtering Checks:

PodFitsResources: Does the node have enough unallocated CPU and RAM to satisfy the Pod's resources.requests?

NodeName / NodeSelector: Does the node match specific labels or the explicit node name specified in the Pod spec?

TaintsAndTolerations: Does the node have a Taint that the Pod lacks a matching Toleration for?

NodeAffinity: Does the node satisfy requiredDuringSchedulingIgnoredDuringExecution rules?

VolumeZone / VolumeNode: Is the required persistent volume (e.g., EBS volume, Azure Disk) physically accessible from this node's availability zone?

Phase 2: Scoring (Priorities)
Ranks all nodes that passed the Filtering phase to determine the absolute best node for the Pod. Each node receives a score (typically 0 to 100), and the node with the highest score is selected.

Common Scoring Rules:

LeastRequestedPriority: Favors nodes with lower CPU/Memory utilization to balance workloads across the cluster.

MostRequestedPriority: Favors nodes with higher utilization (useful for auto-scaling clusters to pack nodes tightly and scale down empty ones).

ImageLocalityPriority: Favors nodes that already have the required container images pre-cached locally.

NodeAffinityPriority / PodTopologySpread: Favors nodes matching preferred affinity rules or spread constraints to maintain high availability.

Phase 3: Binding
Once the winning node is selected, kube-scheduler creates a Binding object via an API call to kube-apiserver, updating the Pod's spec.nodeName. The kubelet on that node then receives the update and spawns the containers.

-----------------------------------------------------

4. Top Interview Questions & Detailed Answers
Q1: What is the main role of kube-scheduler in a Kubernetes cluster?
Answer: The kube-scheduler is responsible for selecting an optimal worker node for unscheduled Pods. It continuously checks for Pods without an assigned node (nodeName: ""), runs them through a Filtering (Predicates) and Scoring (Priorities) pipeline, and binds the winning node to the Pod by updating spec.nodeName via kube-apiserver.

Q2: Does kube-scheduler physically start containers on the worker node?
Answer: No. kube-scheduler only decides where the Pod should run and updates the Pod spec with the selected node's name. The kubelet agent running on that specific worker node detects the assignment, calls the local Container Runtime (containerd), and handles container provisioning.

Q3: What happens when a Pod stays in a Pending state with the event 0/N nodes are available?
Answer: This indicates that all nodes failed the Filtering (Predicates) phase. Common root causes include:

Insufficient Resources: No single node has enough available unallocated CPU or Memory to satisfy the Pod's requests.

Unmatched Taints: Nodes have taints that the Pod lacks matching tolerations for.

Unmatched NodeSelectors/Affinity: No nodes match the required label selectors or affinity constraints in the Pod spec.

Unsatisfied PVC / Volume Binding: The Pod relies on a PersistentVolume bound to an availability zone where no nodes are available.

Q4: What is the difference between Taints/Tolerations and Node Affinity?
Answer:

Node Affinity is a Pod-centric property that attracts Pods to a specific set of nodes based on node labels.

Taints & Tolerations are Node-centric properties where nodes repel Pods. A node with a taint rejects all Pods unless the Pod explicitly has a matching toleration.

Q5: What is Pod Preemption and PriorityClass?
Answer: When a high-priority Pod cannot be scheduled due to a lack of node resources, kube-scheduler can preempt (evict) lower-priority Pods from a node to free up capacity. This behavior is configured using a PriorityClass object, which assigns a numerical value to Pods. High-priority Pods take precedence over low-priority Pods during scheduling bottlenecks.

Q6: How does kube-scheduler handle High Availability (HA)?
Answer: Like kube-controller-manager, kube-scheduler operates in an Active-Passive (Leader-Follower) model using Kubernetes Lease locks. Multiple instances run across control plane nodes, but only one active Leader executes the scheduling algorithm to prevent dual-assignment race conditions. Secondary instances remain idle and take over if the leader fails.

Q7: Can you run multiple schedulers or a custom scheduler in the same cluster?
Answer: Yes. Kubernetes supports Multiple Schedulers. You can deploy a custom scheduler alongside the default kube-scheduler and specify schedulerName: custom-scheduler in a Pod's manifest. The default scheduler ignores Pods assigned to other scheduler names, allowing your custom binary to manage their placement logic.


-----------------------------------------------------------------------

Kubelet :


The **`kubelet`** is the primary, essential worker node agent in Kubernetes. It runs on **every single node** (both Worker Nodes and Control Plane Nodes) inside the cluster. 

While components like the `kube-scheduler` decide *where* a Pod should run, the `kubelet` is the component actually responsible for making sure containers are running, healthy, and configured correctly on the physical or virtual host.

---

## 1. Core Architecture & High-Level Responsibilities

```text
                                [ Control Plane ]
                                       |
                             ( kube-apiserver )
                                       |
                         +-------------+-------------+
                         |  Watches PodSpecs assigned|
                         |  to its specific Node     |
                         v                           v
     +-----------------------------------------------------------------------+
     |                              WORKER NODE                              |
     |                                                                       |
     |   +---------------------------------------------------------------+   |
     |   |                            KUBELET                            |   |
     |   |                                                               |   |
     |   |   1. PodSync Loop       (Reads PodSpec & maintains desired)   |   |
     |   |   2. CNI Interface      (Configures pod network namespaces)   |   |
     |   |   3. CSI Interface      (Mounts volumes/PVs into container)   |   |
     |   |   4. Health Probes      (Liveness, Readiness, Startup)        |   |
     |   |   5. Node Status Sync   (Reports CPU, RAM, & status to API)   |   |
     |   +---------------------------------------------------------------+   |
     |                                   |                                   |
     |                    gRPC / CRI (Container Runtime Interface)           |
     |                                   v                                   |
     |   +---------------------------------------------------------------+   |
     |   |                      CONTAINER RUNTIME                        |   |
     |   |                   (containerd / CRI-O)                        |   |
     |   +---------------------------------------------------------------+   |
     |                                   |                                   |
     |                                   v                                   |
     |                         [ Running Containers ]                        |
     +-----------------------------------------------------------------------+

     
Key Responsibilities :

Pod Lifecycle Management: Translates high-level declarative PodSpecs into low-level runtime container creation, restart, and termination operations.

Node Health & Status Reporting: Periodically sends node status updates, capacity metrics, and health heartbeats back to kube-apiserver.

Health Probing: Executes livenessProbe, readinessProbe, and startupProbe checks against application containers.

Volume & Network Attachment: Interfaces with CSI (Container Storage Interface) to attach/mount storage volumes and CNI (Container Network Interface) to assign IP addresses.

Resource Enforcement (cgroups): Interacts with Linux Kernel cgroups and namespaces to enforce CPU/Memory limits and requests specified in the Pod manifest.


Key Interfaces Managed by kubelet :
The kubelet acts as an orchestrator on the node level by coordinating three standardized plugin interfaces:

CRI : Container Runtime Interface	gRPC protocol used by kubelet to talk to container engines (containerd, CRI-O) without needing hardcoded runtime code.
CNI	: Container Network Interface	Plugin specification used to allocate network interfaces, IP addresses, and routes for newly spawned Pods (e.g., Calico, Cilium, Flannel).
CSI :  Container Storage Interface	Plugin specification used to mount local or cloud storage volumes (AWS EBS, Azure Disk, NFS) directly into container file systems.


3. The Sync Loop: How kubelet Works Internally
The core mechanism of kubelet is an event-driven loop called the Sync Loop (PodSync):

Fetch State: It receives PodSpecs from multiple sources:

kube-apiserver (Primary): Active watch stream of Pods assigned to its node (spec.nodeName == local_node).

Static Pod Directory (Local Files): Inspects /etc/kubernetes/manifests/ for local system manifests (used to bootstrap control plane components like etcd or kube-apiserver).

HTTP Endpoint: Optional URL for fetching remote specs.

Compare State: It queries the local container runtime (via CRI) to inspect what containers are currently running.

Reconcile: If a container died or is missing, kubelet calls CRI to pull the required image and start the container. If a Pod spec changed, it restarts or updates the container accordingly.


4. kubelet Health Checks & Node Eviction
A. Probes Managed by kubelet
Startup Probe: Checks if the application inside the container has started up. All other probes are disabled until this succeeds.

Liveness Probe: Checks if the container is still alive. If this fails, kubelet kills and restarts the container according to its restartPolicy.

Readiness Probe: Checks if the container is ready to accept incoming user traffic. If this fails, kubelet notifies kube-apiserver to remove the Pod IP from Service Endpoints.

B. Node Pressure Eviction
When a node runs low on system resources (disk space, memory, or PID counts), kubelet proactively evicts Pods to prevent node crashes.

[ System Memory/Disk Drop Below Threshold ]
                       |
                       v
     [ Kubelet Triggers Eviction Signal ]
                       |
                       v
     [ Soft/Hard Eviction Threshold Exceeded ]
                       |
                       v
   [ Kubelet Reclaims Space / Evicts Pods ]

   Eviction Thresholds: Configured via flags such as --eviction-hard=memory.available<100Mi,nodefs.available<10%.


Eviction Order: kubelet selects Pods for eviction based on:

Pods exceeding their requested resources (resources.requests).

Pod Quality of Service (QoS) Class:

BestEffort (First to be evicted): Pods with no limits or requests set.

Burstable (Second to be evicted): Pods where requests are lower than limits.

Guaranteed (Last to be evicted): Pods where requests equal limits for both CPU and Memory.


Q1: What is the main difference between kubelet and kube-proxy on a Worker Node?
Answer:

kubelet is responsible for workload lifecycle and node management. It manages container creation, health probes, storage mounting, and reports node status.

kube-proxy is strictly responsible for node-level networking rules. It maintains iptables, IPVS, or eBPF rules to route service traffic to target Pod IPs across the cluster.

Q2: What are Static Pods, and how does the kubelet manage them?
Answer: Static Pods are Pods managed directly by the kubelet on a specific node without passing through the kube-apiserver or kube-scheduler.

The kubelet watches a local host directory (typically /etc/kubernetes/manifests/).

Any YAML file placed in this directory is automatically launched as a Pod by the local kubelet.

Use Case: Used to bootstrap control plane components (kube-apiserver, etcd, kube-scheduler) on self-hosted Kubernetes clusters (e.g., set up via kubeadm).

Q3: How does kubelet communicate with container engines like containerd?
Answer: kubelet communicates with container engines via the Container Runtime Interface (CRI) using gRPC over a UNIX domain socket (e.g., /run/containerd/containerd.sock). This decouples kubelet from specific container engines, allowing it to work seamlessly with containerd, CRI-O, or any CRI-compliant runtime.

Q4: What happens if the kubelet service crashes or stops on a worker node?
Answer:

Container Status: Existing running application containers on that node continue running because container processes are managed independently by the underlying container runtime (containerd).

Loss of Management: kubelet stops sending node heartbeats to kube-apiserver, cannot process new Pod assignments, and stops executing health probes.

Control Plane Response: After node-monitor-grace-period (default ~40s), kube-apiserver marks the node as NotReady. After pod-eviction-timeout (default ~5m), the kube-controller-manager schedules replacement Pods onto other healthy nodes.

Q5: How does kubelet decide which Pods to evict first when a node runs out of Memory (OOM / Node Pressure)?
Answer: kubelet ranks Pods for eviction based on their Quality of Service (QoS) Class and resource usage:

BestEffort Pods (no requests or limits defined) are evicted first.

Burstable Pods (requests < limits) that are exceeding their requested memory are evicted second.

Guaranteed Pods (requests == limits for all containers) are evicted last, and only if system critical processes are threatened.

Q6: What is the difference between a LivenessProbe failure and a ReadinessProbe failure as handled by kubelet?
Answer:

Liveness Probe Failure: kubelet kills the container and restarts it according to the Pod's restartPolicy.

Readiness Probe Failure: kubelet does not kill the container. Instead, it reports to kube-apiserver that the Pod is not ready, causing the Endpoints Controller to remove the Pod's IP from Service endpoints so no new traffic is routed to it until it recovers.

Q7: Where are kubelet logs stored, and how do you troubleshoot a failing node?
Answer: Since kubelet runs directly as a system daemon on the host OS (not inside a container), its logs are inspected using journalctl:

Bash
# Check real-time kubelet logs
journalctl -u kubelet -f

# Check recent kubelet error logs
journalctl -u kubelet -e --no-pager
Troubleshooting steps for a failing node include checking journalctl -u kubelet, verifying CRI runtime status (systemctl status containerd), checking disk usage (df -h), and checking system memory pressure (free -m).


----------------------------------------------------------

Requests vs. Limits : 
# Understanding CPU and Memory Requests vs. Limits in Kubernetes

In Kubernetes, **Requests** and **Limits** are the mechanisms used to control how much CPU and Memory (RAM) a container can consume on a worker node. 

Setting resource requests and limits properly is critical to ensuring application performance, preventing **OOMKilled** (Out Of Memory) crashes, and helping the `kube-scheduler` place workloads efficiently.

---

## 1. Quick Definitions

| Resource Metric | Definition | Analogy |
| :--- | :--- | :--- |
| **`requests`** | The **guaranteed minimum** amount of CPU or Memory Kubernetes reserves for a container. The `kube-scheduler` uses this number to decide which node has enough space to host the Pod. | **Table Reservation:** The minimum size table guaranteed for your party at a restaurant. |
| **`limits`** | The **absolute maximum** amount of CPU or Memory a container is allowed to consume. | **Credit Card Limit:** The hard ceiling you cannot cross, no matter how much you want to spend. |

---

## 2. Resource Units Explained

Before setting values, it is important to understand how Kubernetes measures CPU and Memory:

### A. CPU Measurement (Cores & Millicores)
CPU is measured in **Kubernetes Compute Units** (vCPUs / Cores).
* `1` CPU = 1 vCPU / 1 Core (e.g., 1 AWS vCPU, 1 GCP vCPU, or 1 Hyperthread on bare metal).
* **Millicores (`m`):** CPU is fractional and often expressed in millicores:
  * `1000m` = `1 CPU`
  * `500m` = `0.5 CPU` (50% of 1 CPU core)
  * `250m` = `0.25 CPU` (25% of 1 CPU core)

### B. Memory Measurement (Bytes)
Memory is measured in bytes, commonly expressed using **Binary Megabytes / Gigabytes**:
* **`Mi` (Mebibytes):** $1 \text{ Mi} = 1024 \times 1024 \text{ bytes}$ (Standard base-2 byte notation)
* **`Gi` (Gibibytes):** $1 \text{ Gi} = 1024 \text{ Mi}$
* *(Avoid using `M` or `G` which are decimal/base-10; always use `Mi` and `Gi` in YAMLs).*

---

## 3. Practical Example with YAML Manifest

Here is an example Deployment manifest defining CPU and Memory requests and limits for a web application container:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: api-server
        image: nginx:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"      # 0.25 CPU cores
          limits:
            memory: "512Mi"
            cpu: "500m"      # 0.50 CPU cores


What This Example Means in Practice:
Scheduling (Requests):

The kube-scheduler searches for a node that has at least 250m CPU and 256Mi RAM of unallocated capacity.

If a node has 8 GB of total RAM but 7.8 GB is already reserved by requests from other pods, this new pod will not be scheduled on that node—even if actual current usage is low.

Normal Operation (Between Requests & Limits):

The container starts with a guaranteed 250m CPU and 256Mi RAM.

If traffic spikes, the container can temporarily burst up to 500m CPU and 512Mi RAM as long as spare capacity exists on the host node.

Exceeding Limits (What Happens When Limits Are Hit?):

CPU (Throttled): CPU is a compressible resource. If the container tries to exceed 500m CPU, Kubernetes (via Linux cgroups) will throttle the container's CPU allocation. The application will run slower, but it will not be killed.

Memory (Killed): Memory is an incompressible resource. If the container attempts to allocate more than 512Mi of RAM, the Linux Kernel's Out-Of-Memory killer triggers, and kubelet terminates the container with an OOMKilled (Exit Code 137) status. kubelet will then restart it according to its restartPolicy.


-----------------------------------------------------------
kube-proxy : 


**`kube-proxy`** is a core networking component that runs as a daemon (typically as a `DaemonSet`) on **every single node** inside a Kubernetes cluster. 

While components like the `kubelet` manage container lifecycles and the CNI (Container Network Interface) assigns IP addresses to individual Pods, **`kube-proxy` is strictly responsible for implementing the Kubernetes `Service` abstraction**. It enables virtual IP (ClusterIP) routing and load balancing across a dynamic set of backend Pods.

---

## 1. Core Architecture & High-Level Role

```text
  [ Incoming Client Traffic / Request ]
                   |
                   v
  +-----------------------------------------------------------------------+
  |                              WORKER NODE                              |
  |                                                                       |
  |   +-----------------------+     Watches Services &    +-----------+   |
  |   |    kube-apiserver     | ------------------------> | kube-proxy|   |
  |   +-----------------------+      Endpoints via API    +-----------+   |
  |                                                             |         |
  |                                                             v         |
  |                                                  Translates specs into|
  |                                                  Node Kernel Rules    |
  |                                                             |         |
  |                                                             v         |
  |                                                  +--------------------+
  |                                                  | Kernel Netfilter   |
  |                                                  | (iptables / IPVS)  |
  |                                                  +--------------------+
  |                                                             |         |
  |                                                             v         |
  |  +-----------------------------------------------------------------+  |
  |  |                 Routes directly to Target Pod IPs               |  |
  |  |                                                                 |  |
  |  |      [ Pod A (10.244.1.5) ]      [ Pod B (10.244.1.6) ]           |  |
  |  +-----------------------------------------------------------------+  |
  +-----------------------------------------------------------------------+
Key Responsibilities:

Service Virtual IP (VIP) Routing: Translates abstract, immutable Service Virtual IPs (ClusterIP, NodePort) into the actual, dynamic IP addresses of healthy target Pods.

Cluster Load Balancing: Distributes incoming connections across matching backend Pods for a given Service.

Control Plane Synchronization: Continuously watches the kube-apiserver for updates to Service and Endpoints / EndpointSlice objects.

Kernel Rule Manipulation: Translates Kubernetes Service definitions directly into low-level Linux networking rules (using iptables, IPVS, or eBPF).


2. kube-proxy Proxying Modes
kube-proxy can operate in three main modes. Understanding their differences is crucial for performance tuning and interview discussions:

+----------------------------------------------------+
                  |               KUBE-PROXY MODES                     |
                  +----------------------------------------------------+
                   /                         |                        \
                  v                          v                         v
       +--------------------+      +--------------------+     +--------------------+
       |   User space       |      |      iptables      |     |        IPVS        |
       |  (Legacy/Obsolete) |      | (Default Standard) |     | (High Performance) |
       +--------------------+      +--------------------+     +--------------------+

A. iptables Mode (Default Standard)How it Works: kube-proxy writes sequential iptables rules into the Linux kernel netfilter subsystem for every Service and Endpoint. When traffic hits a ClusterIP, the Linux kernel randomly selects a backend Pod IP based on probability rules.Pros: Stable, widely supported, native to Linux.Cons: Scalability bottlenecks. iptables evaluates rules sequentially ($\mathcal{O}(N)$ complexity). In large clusters with thousands of Services and Pods (e.g., 20,000+ rules), updating or matching iptables rules incurs high CPU overhead and packet processing delays.

B. IPVS Mode (IP Virtual Server - Recommended for Large Clusters)How it Works: Built on the Netfilter hook within the Linux kernel, IPVS uses hash tables ($\mathcal{O}(1)$ complexity) to store routing rules instead of sequential lists.Pros: High performance and minimal latency degradation in large clusters (10,000+ Services). Supports advanced load-balancing algorithms (Round-Robin, Least Connections, Source Hashing, Destination Hashing).Cons: Requires additional Linux kernel modules (ip_vs, ip_vs_rr, etc.) installed on all host worker nodes.

C. Userspace Mode (Legacy / Deprecated)How it Works: kube-proxy opens a port in user space on the host. Traffic travels from kernel space $\rightarrow$ user space (kube-proxy) $\rightarrow$ kernel space $\rightarrow$ target Pod.Cons: Extremely slow due to continuous context switching between kernel space and user space for every packet. Obsolete in modern clusters.💡 Modern Alternative (eBPF / Cilium): Many modern platforms bypass kube-proxy entirely using eBPF-based CNI plugins like Cilium (kube-proxy-replacement=true), which route packets directly within kernel socket layers with zero iptables overhead.


3. How kube-proxy Handles Service Types

ClusterIP   	Creates internal kernel rules mapping the Service IP (e.g., 10.96.0.10:80) to active Pod IPs (e.g., 10.244.1.15:8080).

NodePort	Opens a high-range port (30000–32767) on every Worker Node's physical IP address and creates routing rules directing that port to the underlying ClusterIP and Pod endpoints.

LoadBalancer	Relies on cloud-controller-manager to provision an external cloud load balancer (e.g., AWS ALB/NLB), which forwards incoming external traffic to the NodePort rules maintained by kube-proxy.


---------------------------------------------------
Top DevOps & Platform Engineering Interview Questions
Q1: What is the primary role of kube-proxy in a Kubernetes cluster?
Answer: kube-proxy is a network agent running on each node that implements the Kubernetes Service abstraction. It watches the kube-apiserver for changes to Services and Endpoints/EndpointSlices, updating local Linux kernel routing tables (iptables or IPVS) to route traffic sent to virtual IPs (ClusterIP, NodePort) directly to healthy backend Pod IPs.


Q2: What is the difference between kube-proxy and a CNI (Container Network Interface) plugin like Calico or Flannel?
Answer:

CNI (Network Plugin): Responsible for Pod-to-Pod connectivity. It creates network interfaces, assigns IP addresses to Pods, and ensures Pod A on Node 1 can ping Pod B on Node 2.

kube-proxy: Responsible for Service-to-Pod load balancing. It manages the virtual layer above Pods, taking traffic sent to a static Service IP and routing/load-balancing it across dynamic, short-lived Pod IPs provided by the CNI.


Q3: Why does iptables mode in kube-proxy suffer from scaling issues in large clusters?Answer: iptables rules are evaluated sequentially ($\mathcal{O}(N)$ lookup time). As a cluster grows to thousands of Services and tens of thousands of Pod endpoints:Every packet must evaluate a massive list of sequential rules, causing CPU overhead and network latency.Every time a single Pod is created or deleted, kube-proxy must rewrite and re-read large chunks of the iptables rule table, causing heavy lock contention in the Linux kernel.


Q4: How does IPVS mode overcome the limitations of iptables mode?Answer: IPVS uses hash tables ($\mathcal{O}(1)$ lookup time) instead of sequential lists. Packet lookup speed remains constant regardless of whether the cluster has 10 Services or 10,000 Services. Furthermore, IPVS supports advanced load balancing algorithms (like Least Connections or Source Hashing), whereas iptables only supports basic random probability.

Q5: What is externalTrafficPolicy: Local and how does it affect kube-proxy behavior?
Answer: By default (externalTrafficPolicy: Cluster), when traffic hits a NodePort on Node A, kube-proxy may route that traffic to a Pod running on Node B, incurring an extra network hop (SNAT/cross-node latency).

When set to Local, kube-proxy forces the node to route traffic only to Pods running locally on that exact node.

Pros: Preserves the client's real source IP address (no SNAT) and eliminates extra network hops.

Cons: If Node A has no running local Pods for that Service, traffic sent to Node A is dropped.


Q6: What happens if kube-proxy crashes or stops running on a worker node?
Answer:

Existing Traffic: Already established network connections and existing iptables/IPVS rules stored in the host Linux kernel continue to work temporarily.

Dynamic Breakage: Because kube-proxy is inactive, it stops receiving updates from kube-apiserver. If Pods scale up, scale down, or crash, kernel routing rules will not be updated, leading to traffic being sent to dead Pod IPs or new Pods being ignored


Q7: Can a Kubernetes cluster run without kube-proxy?
Answer: Yes. Modern eBPF-based networking solutions like Cilium can run in kube-proxy-replacement mode. They hook directly into Linux kernel sockets using eBPF programs, intercepting and routing Service traffic with higher performance and lower overhead than kube-proxy.