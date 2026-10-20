# Kubernetes Basics

## What Is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform used to deploy, manage, scale, and maintain containerized applications.

It helps automate tasks such as:

- Deploying containerized applications.
- Maintaining the desired number of application replicas.
- Restarting containers when required.
- Exposing applications through network Services.
- Managing application configuration and storage.
- Updating applications with controlled rollouts.

Kubernetes manages the desired state of applications by continuously comparing the actual cluster state with the declared configuration.

--------------------------------------------------------------------

## Why Do We Need Kubernetes?

Running containers individually with Docker works well for many workloads. However, as applications grow, managing multiple containers across machines becomes more complex.

Kubernetes helps with:

- **Orchestration:** Managing application workloads across cluster nodes.
- **Self-healing:** Recreating or restarting workloads when supported by their controllers and configuration.
- **Scaling:** Increasing or decreasing the number of application replicas.
- **Service discovery:** Allowing workloads to communicate through stable Service names.
- **Declarative management:** Defining the desired state using YAML manifests.
- **Rolling updates:** Updating application versions while controlling disruption.

Kubernetes does not eliminate the need for monitoring, backups, security configuration, or operational planning.

--------------------------------------------------------------------

## What Is a Kubernetes Cluster?

A Kubernetes cluster is a group of machines that work together to run containerized applications.

A cluster consists of two main parts:

1. **Control Plane:** Makes cluster-level decisions and manages the desired state.
2. **Worker Nodes:** Run application workloads.

A local learning cluster can run on a single worker node, while production clusters commonly use multiple nodes for improved capacity and resilience.

--------------------------------------------------------------------

## Control Plane Components

### 1. API Server

The API server is the main entry point for Kubernetes API operations.

- Receives requests from `kubectl` and other clients.
- Validates and processes API requests.
- Exposes Kubernetes resources through the API.
- Coordinates access to cluster state.

### 2. etcd

`etcd` is a distributed key-value store that holds Kubernetes cluster data.

- Stores persistent cluster state and configuration.
- Is used by the API server to read and write cluster data.
- Requires reliable backup and recovery planning in production.

### 3. Scheduler

The scheduler selects a suitable node for Pods that have not yet been assigned to a node.

It considers scheduling constraints and available resources, including CPU and memory requests, node selectors, affinity rules, and taints and tolerations.

### 4. Controller Manager

The controller manager runs controllers that continually compare the observed state with the desired state.

For example, a Deployment's controllers help maintain the requested number of Pod replicas through ReplicaSets.

--------------------------------------------------------------------

## Worker Node Components

### 1. Kubelet

The kubelet runs on each node and ensures that the Pods assigned to that node are handled according to their specifications.

It communicates with the API server and works with the configured container runtime.

### 2. Container Runtime

The container runtime runs containers. Kubernetes uses a compatible runtime through the Container Runtime Interface (CRI).

In the local learning environment, the cluster uses k3d with k3s. Docker is used as part of the local container and cluster environment; this does not mean Docker Engine itself must be the Kubernetes runtime inside every node.

### 3. Networking Components

Kubernetes networking enables communication between Pods, Services, and external clients.

Service networking is implemented by the cluster's networking components. In many clusters, kube-proxy handles Service networking; other implementations can provide this functionality differently.

--------------------------------------------------------------------

## Kubernetes Objects and Desired State

A Kubernetes object represents a resource that Kubernetes manages, such as a Pod, Deployment, Service, ConfigMap, or PersistentVolumeClaim.

A declarative manifest describes the desired state of a resource. Kubernetes controllers work to reconcile the actual state toward that desired state.

For example, a Deployment configured with three replicas expresses the desired state that three application Pod replicas should be maintained.

### Common Manifest Fields

| Field | Purpose |
|---|---|
| `apiVersion` | Identifies the API version used by the resource. |
| `kind` | Specifies the resource type, such as `Pod` or `Deployment`. |
| `metadata` | Contains resource information such as its name and labels. |
| `spec` | Describes the desired configuration. |
| `status` | Reports the resource's observed state; normally populated by Kubernetes. |

--------------------------------------------------------------------

## YAML Manifests

Kubernetes resources are commonly configured using YAML manifests.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
```

Important points:

- `apiVersion: v1` identifies the core API version used by this Pod.
- `kind: Pod` specifies that the resource is a Pod.
- `metadata.name` sets the Pod's name.
- `metadata.labels` assigns labels that can be used to identify and select resources.
- `spec.containers` defines the containers in the Pod.
- `image` specifies the container image.
- `containerPort` documents the port the container is expected to listen on; it does not by itself expose the application outside the Pod.

YAML indentation matters. Use spaces consistently and keep child fields correctly indented.

--------------------------------------------------------------------

## Essential kubectl Commands

`kubectl` is the command-line tool used to communicate with a Kubernetes cluster through the API server.

```bash
# Display cluster nodes
kubectl get nodes

# Display Pods in the current namespace
kubectl get pods

# Display Pods across all namespaces
kubectl get pods -A

# Display Deployments
kubectl get deployments

# Display Services
kubectl get services

# Inspect a specific resource
kubectl describe pod nginx-pod

# Apply a manifest
kubectl apply -f pod.yaml

# Delete a resource using its manifest
kubectl delete -f pod.yaml

# Display the current cluster context
kubectl config current-context
```

Use `kubectl describe` and resource events to investigate why a resource is not reaching its expected state.

--------------------------------------------------------------------

## Key Takeaways

- Kubernetes orchestrates and manages containerized applications.
- A cluster consists of a control plane and worker nodes.
- The API server provides the main interface to the Kubernetes API.
- `etcd` stores cluster state.
- The scheduler assigns pending Pods to suitable nodes.
- Controllers reconcile actual state with desired state.
- The kubelet manages Pods assigned to its node.
- YAML manifests describe resources declaratively.
- `kubectl` is the primary command-line interface for cluster operations.
- A Pod's `containerPort` declaration alone does not expose the application externally.

--------------------------------------------------------------------

## Revision Questions

1. What is Kubernetes, and why is orchestration useful?
2. What is the difference between the control plane and a worker node?
3. What roles do the API server, etcd, scheduler, and controller manager perform?
4. What is the purpose of the kubelet?
5. What is declarative configuration?
6. What do `apiVersion`, `kind`, `metadata`, and `spec` represent?
7. Does defining `containerPort: 80` expose a Pod outside the cluster?
8. What is the difference between `kubectl get` and `kubectl describe`?