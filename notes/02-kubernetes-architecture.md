# Kubernetes Architecture

## Overview

Kubernetes follows a cluster-based architecture consisting of a **control plane** and **worker nodes**.

- **Control Plane:** Manages the cluster, makes scheduling decisions, and maintains the desired state.
- **Worker Nodes:** Run application Pods and provide the resources required by workloads.

The components communicate through the Kubernetes API, primarily exposed by the API server.

--------------------------------------------------------------------

## Kubernetes Architecture Diagram

```text
                    Kubernetes Cluster
                           |
             +-------------+-------------+
             |                           |
      CONTROL PLANE                 WORKER NODE
             |                           |
      +------+-------+          +--------+--------+
      |      |       |          |        |        |
   API     etcd  Scheduler   Kubelet  Runtime  Networking
  Server                        |        |        |
      |                         +----+---+--------+
 Controller                             |
  Manager                          Application Pods
```

This is a simplified conceptual diagram. Actual component placement and networking implementations depend on the cluster configuration.

--------------------------------------------------------------------

## Control Plane Components

### 1. API Server (`kube-apiserver`)

The API server is the central interface for Kubernetes API operations.

Responsibilities:

- Accepts requests from `kubectl`, controllers, and other clients.
- Authenticates and authorizes requests.
- Validates resource configurations.
- Reads and updates cluster state stored in etcd.
- Exposes the Kubernetes API to other components.

Example:

```bash
kubectl get nodes
```

When this command runs, `kubectl` communicates with the API server to retrieve node information.

### 2. etcd

`etcd` is a distributed key-value store used to persist Kubernetes cluster data.

Responsibilities:

- Stores Kubernetes resource state and configuration.
- Provides persistent storage for the cluster's control-plane data.
- Supports the API server's state-management operations.

Important points:

- etcd is a critical component for cluster recovery.
- Production environments require appropriate backup, access control, and recovery procedures.
- etcd is not the storage location for application files written inside a Pod's mounted volume.

### 3. Scheduler (`kube-scheduler`)

The scheduler selects a suitable node for a newly created Pod that has not yet been assigned to one.

It evaluates factors such as:

- Available node resources and Pod resource requests.
- Node selectors and affinity rules.
- Taints and tolerations.
- Other scheduling constraints.

The scheduler selects a node; the kubelet on that node then works to ensure the Pod's containers are running.

### 4. Controller Manager (`kube-controller-manager`)

The controller manager runs controllers that continuously reconcile actual cluster state with desired state.

Examples:

- **Deployment and ReplicaSet controllers:** Help maintain the requested number of Pod replicas.
- **Node controller:** Monitors node status and responds to node-related changes.
- **Job controller:** Manages Kubernetes Job workloads.

Controllers generally work through the API server rather than directly managing every workload themselves.

--------------------------------------------------------------------

## Worker Node Components

### 1. Kubelet

The kubelet is the node agent responsible for ensuring that Pods assigned to its node are handled according to their specifications.

Responsibilities:

- Observes Pod specifications assigned to the node.
- Works with the container runtime to start and manage containers.
- Reports node and Pod status to the API server.
- Performs configured container health checks and responds to their results.

### 2. Container Runtime

The container runtime runs containers on the worker node.

Kubernetes communicates with a compatible runtime through the **Container Runtime Interface (CRI)**.

The runtime is responsible for tasks such as pulling images and running containers.

In this learning environment, the Kubernetes cluster was created using **k3d with k3s**, with Docker used to run the local cluster environment. Docker on the host should not be confused with the specific runtime used inside every Kubernetes node.

### 3. Networking

Kubernetes networking enables communication between Pods, Services, nodes, and external clients.

Key concepts:

- Pods receive network connectivity through the cluster networking implementation.
- Services provide stable access to selected workloads.
- DNS enables workloads to discover Services by name.
- Network traffic is handled by the cluster's networking components.

Many clusters use kube-proxy for Service networking, while some use alternative implementations that provide equivalent functionality.

--------------------------------------------------------------------

## How a Kubernetes Request Flows

Consider creating a Deployment using a YAML manifest.

### Step 1: Submit the Configuration

You run:

```bash
kubectl apply -f nginx-deployment.yaml
```

`kubectl` sends the request to the Kubernetes API server.

### Step 2: Validate and Store the Desired State

The API server processes the request, including authentication, authorization, and validation.

The accepted resource state is persisted in etcd.

### Step 3: Controllers Reconcile the Deployment

The Deployment controller observes the new Deployment and ensures that the required ReplicaSet is created or updated.

The ReplicaSet controller works to maintain the requested number of Pods.

### Step 4: The Scheduler Selects a Node

New Pods without an assigned node are evaluated by the scheduler.

The scheduler selects a suitable node based on resource availability and scheduling constraints.

### Step 5: The Kubelet Starts the Workload

The kubelet on the selected node observes the assigned Pod specification.

It works with the container runtime to pull the required image when necessary and start the container.

### Step 6: Kubernetes Reports Status

The kubelet reports relevant Pod and node status through the API server.

Kubernetes continues reconciling the workload toward its desired state.

**Important:** Accepting a manifest does not guarantee that an application will become healthy. Image-pull failures, scheduling constraints, configuration errors, or failed health checks can prevent a workload from becoming ready.

--------------------------------------------------------------------

## Desired State and Reconciliation

Kubernetes is designed around declarative management.

For example, a Deployment may specify:

```yaml
spec:
  replicas: 3
```

This declares the desired number of replicas.

If only two replicas are available, the relevant controllers attempt to restore the desired state, provided the cluster has the necessary capacity and the workload can be created successfully.

The process of repeatedly comparing actual state with desired state and acting to reduce the difference is called **reconciliation**.

--------------------------------------------------------------------

## High Availability and Failure Considerations

A production cluster must consider failures of both control-plane components and worker nodes.

### Control Plane Availability

- A single control-plane instance can become a single point of failure.
- Highly available control planes commonly use multiple control-plane instances.
- etcd requires its own availability and backup strategy.

### Worker Node Availability

- If a worker node fails, the Pods on that node may become unavailable.
- Controllers can create replacement Pods, but scheduling them requires suitable healthy nodes and sufficient capacity.
- Running several replicas on one node does not protect an application against failure of that node.
- Spreading replicas across nodes can improve resilience.

A multi-node cluster improves the available recovery options, but it does not automatically guarantee application availability.

--------------------------------------------------------------------

## Commands for Inspecting Architecture

```bash
# List cluster nodes
kubectl get nodes -o wide

# Inspect node conditions, capacity, and allocated resources
kubectl describe node

# View system Pods
kubectl get pods -n kube-system

# View Pods across all namespaces
kubectl get pods -A

# Display cluster information
kubectl cluster-info

# Inspect a Deployment
kubectl describe deployment nginx-deployment

# View recent cluster events
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Some control-plane components may run as static Pods, while others can be managed differently depending on the Kubernetes distribution. A local k3s-based cluster may not expose every component in the same way as a kubeadm-installed cluster.

--------------------------------------------------------------------

## Key Takeaways

- Kubernetes architecture separates cluster management from workload execution.
- The API server is the main interface to the Kubernetes API.
- etcd persists cluster state.
- The scheduler assigns pending Pods to suitable nodes.
- Controllers reconcile the actual state toward the desired state.
- The kubelet manages Pods assigned to its node.
- The container runtime executes containers.
- Services and cluster networking enable workload communication.
- Deployments use ReplicaSets to maintain the desired number of Pods.
- Production resilience requires planning for node, control-plane, and storage failures.

--------------------------------------------------------------------

## Revision Questions

1. What are the two main parts of Kubernetes cluster architecture?
2. What role does the API server play?
3. What information does etcd store?
4. What is the difference between the scheduler and the kubelet?
5. How do controllers help maintain the desired state?
6. How does a Deployment create and maintain Pod replicas?
7. What happens after you run `kubectl apply -f nginx-deployment.yaml`?
8. Why is a multi-node cluster useful for application resilience?
9. Does creating multiple replicas on one node protect against that node failing?
10. Why might a Deployment exist even when its application Pods are not ready?