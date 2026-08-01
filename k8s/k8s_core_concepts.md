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