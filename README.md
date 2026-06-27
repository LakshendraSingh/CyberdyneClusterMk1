# Kubernetes Cluster Architecture Overview

This document provides a detailed breakdown of the **Kubernetes (K8s) architecture** based on the accompanying architecture diagram. It explains the core components, their responsibilities, and how they interact to orchestrate containerized applications at scale.

---

# Architecture Diagram Overview

The diagram illustrates a standard Kubernetes cluster architecture, which is divided into three primary zones:

* **Admin / Client Machine** – The control interface where administrators and operators interact with the cluster.
* **Control Plane Node** – The brain of the cluster that manages cluster state, scheduling, and orchestration.
* **Worker Nodes** – The execution layer that runs application workloads inside Pods.

These components communicate through the **Cluster Network**, enabling secure and reliable communication across the cluster.

```text
                        +----------------------------+
                        |   Admin / Client Machine   |
                        | (kubectl, cluster management)|
                        +--------------+-------------+
                                       |
                               kubectl | (REST API)
                                       v
+------------------------------------------------------------------------+
|                          CONTROL PLANE NODE                            |
|                                                                        |
|  +------------------+  +------------------+  +-----------------------+ |
|  |    API Server    |<-|    Scheduler     |  |  Controller Manager   | |
|  +--------+---------+  +------------------+  +-----------------------+ |
|           |                                                            |
|           v                                                            |
|  +------------------+                                                  |
|  |       etcd       |                                                  |
|  +------------------+                                                  |
+------------------------------------+-----------------------------------+
                                     |
                         Manage &    |
                         Schedule    |
                         Workloads   |
                                     v
+------------------------------------+-----------------------------------+
|                            WORKER NODES                                |
|                                                                        |
|  +----------------------------------+  +-----------------------------+ |
|  |          WORKER NODE 1           |  |        WORKER NODE 2        | |
|  |                                  |  |                             | |
|  |  +---------+  +---------------+  |  |  +---------+  +-----------+ | |
|  |  | Kubelet |  |  Kube-Proxy   |  |  |  | Kubelet |  |Kube-Proxy | | |
|  |  +---------+  +---------------+  |  |  +---------+  +-----------+ | |
|  |  +----------------------------+  |  |  +------------------------+ | |
|  |  |     Container Runtime      |  |  |  |   Container Runtime    | | |
|  |  +----------------------------+  |  |  +------------------------+ | |
|  |                                  |  |                             | |
|  |  [Pod]       [Pod]       [Pod]   |  |  [Pod]      [Pod]     [Pod] | |
|  +----------------------------------+  +-----------------------------+ |
+------------------------------------------------------------------------+
```

---

# Component Breakdown

## 1. Admin / Client Machine

The entry point for any administrator, operator, or CI/CD system interacting with the Kubernetes cluster.

### Responsibilities

* Manage cluster topology and configuration
* Deploy applications using YAML manifests
* Define the desired state of applications
* Monitor cluster health
* Debug issues and collect logs

### Key Tool

#### `kubectl`

The official Kubernetes CLI used to communicate with the Control Plane through REST API requests.

---

## 2. Control Plane Node

The Control Plane manages the entire Kubernetes cluster. It continuously monitors cluster health, schedules workloads, and reconciles the desired and actual states.

### API Server (`kube-apiserver`)

The **frontend** of Kubernetes.

#### Responsibilities

* Exposes the Kubernetes REST API
* Acts as the single entry point for all cluster operations
* Coordinates communication between all control plane components

#### Interaction

* Receives requests from `kubectl`
* Validates requests
* Persists cluster state in **etcd**

---

### Scheduler (`kube-scheduler`)

The **matchmaker** of Kubernetes.

#### Responsibilities

* Watches for newly created Pods without assigned nodes
* Selects the best Worker Node for each Pod

#### Scheduling Criteria

* CPU availability
* Memory availability
* Resource requests and limits
* Node affinity / anti-affinity
* Hardware and software constraints
* Data locality

---

### Controller Manager (`kube-controller-manager`)

The **regulator** of the cluster.

#### Responsibilities

Runs multiple controller processes that continuously compare:

* Desired cluster state
* Actual cluster state

Whenever differences are detected, controllers reconcile them automatically.

#### Examples

* Node Controller
* Replication Controller
* EndpointSlice Controller

---

### etcd

The **distributed cluster database**.

#### Responsibilities

* Stores all cluster configuration
* Stores desired application state
* Stores metadata
* Stores secrets
* Stores cluster events

> **Note:** All writes to `etcd` occur through the API Server to maintain consistency and integrity.

---

## 3. Worker Nodes

Worker Nodes execute the application workloads by running Pods.

A cluster may contain one or many worker nodes.

---

### Kubelet

The **node agent**.

#### Responsibilities

* Receives Pod specifications from the Control Plane
* Ensures containers are running correctly
* Reports node health
* Reports resource utilization
* Restarts failed containers when required

---

### Kube-Proxy (`kube-proxy`)

The **network proxy**.

#### Responsibilities

* Maintains networking rules
* Enables Pod communication
* Routes incoming traffic
* Performs load balancing across Pods

#### Supported Protocols

* TCP
* UDP
* SCTP

---

### Container Runtime

The **container execution engine**.

Responsible for:

* Pulling container images
* Starting containers
* Stopping containers
* Managing container lifecycle

#### Common Container Runtimes

* `containerd`
* `CRI-O`

---

### Pods

The smallest deployable unit in Kubernetes.

A Pod contains:

* One or more containers
* Shared networking
* Shared storage volumes
* Runtime specifications

Pods are scheduled as a single unit onto Worker Nodes.

---

## 4. Cluster Network

The virtual networking layer connecting every component in the Kubernetes cluster.

### Responsibilities

* Assigns a unique IP address to every Pod
* Enables Pod-to-Pod communication
* Enables Node-to-Node communication
* Enables Service-to-Pod communication
* Supports Network Policies for traffic control
* Eliminates the need for explicit port mapping between Pods

---

# How a Deployment Works

The following example demonstrates the lifecycle of deploying a simple web application.

## Step 1: Submission

An administrator deploys an application:

```bash
kubectl apply -f deployment.yaml
```

The request is sent to the **API Server**.

---

## Step 2: Validation and Storage

The API Server:

1. Validates the deployment manifest
2. Stores the desired state in **etcd**

---

## Step 3: Reconciliation

The Controller Manager detects:

* Desired replicas = **3**
* Current replicas = **0**

It requests the API Server to create **three Pod definitions**.

---

## Step 4: Scheduling

The Scheduler observes three pending Pods.

It evaluates each Worker Node based on:

* Available CPU
* Available memory
* Scheduling constraints

The Scheduler assigns each Pod to the most suitable Worker Node.

---

## Step 5: Execution

The Kubelet on each selected Worker Node:

1. Detects the assigned Pod
2. Pulls the required container image
3. Starts the container using the local Container Runtime

---

## Step 6: Networking

`kube-proxy` configures networking rules so that:

* Services can reach the new Pods
* External clients can access the application
* Traffic is load balanced across replicas

---

## Step 7: Health Reporting

The Kubelet continuously monitors:

* Container health
* Pod status
* Resource usage

It reports this information back to the API Server, allowing the Control Plane to maintain the desired state and recover automatically from failures.

---

# Summary

The Kubernetes architecture follows a clear separation of responsibilities:

| Component              | Primary Responsibility                              |
| ---------------------- | --------------------------------------------------- |
| **Admin Machine**      | User interaction through `kubectl`                  |
| **API Server**         | Entry point for all cluster operations              |
| **Scheduler**          | Assigns Pods to Worker Nodes                        |
| **Controller Manager** | Maintains desired cluster state                     |
| **etcd**               | Stores cluster state and configuration              |
| **Worker Nodes**       | Execute application workloads                       |
| **Kubelet**            | Runs and monitors Pods                              |
| **Kube-Proxy**         | Handles networking and traffic routing              |
| **Container Runtime**  | Runs containers                                     |
| **Pods**               | Smallest deployable application unit                |
| **Cluster Network**    | Enables secure communication throughout the cluster |
