# Pods in Kubernetes

## What Is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It represents one or more containers that run together and share certain resources.

A Pod provides:

- A shared network environment for its containers.
- The ability to share storage volumes.
- A specification describing how its containers should run.

Most application Pods contain one container. Multiple containers can be placed in the same Pod when they need to work closely together, such as an application container and a supporting sidecar.

**Important:** Kubernetes manages Pods as units. Containers inside a Pod share the Pod's network namespace, including its IP address and port space.

--------------------------------------------------------------------

## Why Does Kubernetes Use Pods?

Kubernetes uses Pods as the abstraction for running application workloads on cluster nodes.

Pods allow Kubernetes to:

- Schedule workloads onto suitable nodes.
- Run containers with shared networking and storage.
- Monitor container and Pod status.
- Replace failed or terminated workloads when a controller is responsible for maintaining them.

A Pod is not the same as a container. A container is a running instance of an image, while a Pod is the Kubernetes resource that groups and manages one or more containers.

--------------------------------------------------------------------

## Pod Architecture

```text
                  Kubernetes Cluster
                          |
                    Worker Node
                          |
                    +-----------+
                    |    Pod    |
                    |           |
                    | Container |
                    |  (Nginx)  |
                    |           |
                    | Pod IP    |
                    | Volumes   |
                    +-----------+
```

This is a simplified illustration of a single-container Pod.

Containers within the same Pod share its network namespace. They can communicate through `localhost`, provided they use different ports when listening on the same network interface.

--------------------------------------------------------------------

## Creating a Pod with YAML

During the practical exercises, an Nginx Pod was created using the following manifest.

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

### Explanation of the Manifest

| Field | Purpose |
|---|---|
| `apiVersion: v1` | Uses the core Kubernetes API version for Pods. |
| `kind: Pod` | Declares the resource type. |
| `metadata.name` | Assigns the name `nginx-pod`. |
| `metadata.labels` | Adds the label `app: nginx`. |
| `spec.containers` | Defines the containers in the Pod. |
| `name` | Names the container within the Pod. |
| `image` | Specifies the container image, `nginx:stable`. |
| `containerPort` | Documents the port expected to be used by the container. |

**Note:** `containerPort: 80` does not itself publish the port outside the Pod. External or stable access typically requires an appropriate networking configuration, such as a Service.

--------------------------------------------------------------------

## Creating and Inspecting a Pod

### 1. Create the Pod

```bash
kubectl apply -f pod.yaml
```

Kubernetes creates or updates the resource declared in the manifest.

### 2. List Pods

```bash
kubectl get pods
```

Show additional details:

```bash
kubectl get pods -o wide
```

The wider output includes details such as the Pod IP and the node on which it is scheduled.

### 3. Inspect a Pod

```bash
kubectl describe pod nginx-pod
```

This displays information including:

- Pod status and conditions.
- Container image and state.
- Node assignment and Pod IP.
- Events that may explain failures.

### 4. View Container Logs

```bash
kubectl logs nginx-pod
```

For a Pod with multiple containers, specify the container:

```bash
kubectl logs nginx-pod -c nginx
```

### 5. Execute a Command Inside a Container

```bash
kubectl exec nginx-pod -- nginx -v
```

Open an interactive shell when the container image includes a suitable shell:

```bash
kubectl exec -it nginx-pod -- sh
```

--------------------------------------------------------------------

## Practical Exercise: Deploy an Nginx Pod

The following commands were used during the learning exercise.

### Step 1: Apply the Manifest

```bash
kubectl apply -f day-02/nginx-pod.yaml
```

### Step 2: Verify the Pod

```bash
kubectl get pods
kubectl describe pod nginx-pod
```

The Pod reached the `Running` phase.

### Step 3: Test the Application

A request was made from inside the cluster to verify that Nginx responded successfully.

The HTTP response returned status **200 OK**, confirming that the test request succeeded.

### Step 4: Understand the Pod IP

A Pod receives an IP address from the cluster's networking implementation. That address can change when the Pod is replaced, so applications should generally use Services rather than rely on Pod IPs directly.

--------------------------------------------------------------------

## Pod Lifecycle and Status

Kubernetes reports Pod phase and container state information to help operators understand workload status.

Common Pod phases include:

| Phase | Meaning |
|---|---|
| `Pending` | The Pod has been accepted, but one or more containers have not finished setup and started. |
| `Running` | The Pod has been bound to a node, and its containers have been created. At least one container is running or starting/restarting. |
| `Succeeded` | All containers have terminated successfully and will not be restarted. |
| `Failed` | All containers have terminated, and at least one terminated unsuccessfully. |
| `Unknown` | Kubernetes cannot determine the Pod's state, commonly due to a communication problem with the node. |

`CrashLoopBackOff` is **not** a Pod phase. It is a displayed container waiting reason indicating that Kubernetes is delaying repeated restart attempts.

A Pod in the `Running` phase is not necessarily ready to serve traffic. Check the `READY` column and Pod conditions as well.

--------------------------------------------------------------------

## Labels and Selectors

Labels are key-value pairs attached to Kubernetes objects. They help organize resources and allow other resources to select matching Pods.

Example:

```yaml
metadata:
  labels:
    app: nginx
    environment: development
```

List Pods with a specific label:

```bash
kubectl get pods -l app=nginx
```

Show labels in the output:

```bash
kubectl get pods --show-labels
```

Labels are especially important for Deployments and Services, which use selectors to identify the Pods they manage or route traffic to.

--------------------------------------------------------------------

## Standalone Pods vs. Deployments

A standalone Pod is created directly as a Pod resource. Kubernetes does not automatically create a replacement merely because that standalone Pod is deleted.

A Deployment manages ReplicaSets, which maintain the requested number of Pod replicas.

| Feature | Standalone Pod | Deployment |
|---|---|---|
| Desired replica count | Not managed by a Deployment | Declared through `spec.replicas` |
| Replacement after Pod deletion | No automatic replacement controller for that Pod | Controller works to create a replacement |
| Rolling updates | Not provided as a Deployment workflow | Supported |
| Recommended for typical replicated applications | Usually not preferred | Commonly preferred |

Even with a Deployment, a replacement Pod may take time to schedule and become ready. Recovery depends on cluster capacity, configuration, and workload health.

--------------------------------------------------------------------

## Deleting a Pod

Delete a Pod using its manifest:

```bash
kubectl delete -f pod.yaml
```

Or delete it by name:

```bash
kubectl delete pod nginx-pod
```

Deleting a standalone Pod removes that Pod resource. If the Pod is managed by a Deployment or another controller, that controller may create a replacement to restore the desired state.

--------------------------------------------------------------------

## Key Takeaways

- A Pod is the smallest deployable unit in Kubernetes.
- A Pod contains one or more containers.
- Containers in the same Pod share networking and can share mounted storage.
- A Pod's IP address is not a stable application endpoint.
- `containerPort` documents a container port but does not expose it by itself.
- Labels and selectors help Kubernetes resources identify matching Pods.
- `kubectl get`, `describe`, `logs`, and `exec` are essential Pod inspection tools.
- A standalone Pod is not automatically replaced by a Deployment controller.
- Deployments are generally preferred for replicated, long-running application workloads.
- `Running` does not necessarily mean that a Pod is ready to serve traffic.

--------------------------------------------------------------------

## Revision Questions

1. What is a Pod, and how does it differ from a container?
2. What resources can containers within a Pod share?
3. What does `containerPort: 80` actually do?
4. Why should applications avoid depending directly on Pod IP addresses?
5. What is the difference between `kubectl get pods` and `kubectl describe pod`?
6. How can you inspect a container's logs?
7. What is the difference between a standalone Pod and a Deployment-managed Pod?
8. Is `CrashLoopBackOff` a Pod phase?
9. Does a Pod in the `Running` phase necessarily mean it is ready?
10. How do labels and selectors help Services and Deployments?