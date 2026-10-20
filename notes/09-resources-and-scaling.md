# Resources and Scaling

## Why Do Resources Matter?

Kubernetes workloads run on nodes with finite CPU, memory, and other resources. If workloads request too few resources, applications may perform poorly. If they request too many, the scheduler may struggle to place Pods even when some node capacity remains.

Kubernetes provides resource requests, limits, resource monitoring, and scaling mechanisms to help workloads use available capacity effectively.

Key concepts include:

- CPU and memory requests.
- CPU and memory limits.
- Quality of Service (QoS) classes.
- Horizontal Pod Autoscaler (HPA).
- Vertical Pod Autoscaler (VPA).
- Cluster Autoscaler.
- Metrics and resource troubleshooting.

--------------------------------------------------------------------

## CPU and Memory Requests

A **resource request** specifies the amount of a resource a container is expected to need for scheduling purposes.

The Kubernetes scheduler uses the requests of containers in a Pod when deciding whether a node has sufficient allocatable resources to run it.

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
```

This requests:

- `250m` CPU, equal to one-quarter of a CPU core.
- `128Mi` memory.

Requests are used for scheduling decisions. They are not a guarantee that an application will always receive exactly that amount of actual usage.

--------------------------------------------------------------------

## CPU and Memory Limits

A **resource limit** specifies the maximum resource consumption permitted for a container.

Example:

```yaml
resources:
  limits:
    cpu: "1"
    memory: "512Mi"
```

This configures a limit of one CPU core and 512 mebibytes of memory.

### CPU Limits

When a container attempts to use more CPU than its configured limit, it may be throttled.

CPU throttling can reduce application performance, especially for workloads that need short bursts of CPU capacity.

### Memory Limits

Memory limits are enforced differently. If a container exceeds its memory limit, it may be terminated by the system, commonly with an out-of-memory (OOM) condition.

A container terminated for exceeding its memory limit may restart, depending on the Pod's restart policy and workload controller.

**Important:** Set requests and limits based on observed workload behaviour and testing rather than choosing arbitrary values.

--------------------------------------------------------------------

## Example: Requests and Limits in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: nginx:stable
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

In this example:

- The container requests 250 millicores of CPU and 128Mi of memory.
- The container has a CPU limit of 500 millicores and a memory limit of 256Mi.
- The scheduler uses the requests when evaluating whether the Pod fits on a node.

--------------------------------------------------------------------

## Understanding CPU Units

Kubernetes CPU resources are measured in CPU units.

| Value | Meaning |
|---|---|
| `1` | One CPU core |
| `500m` | Half a CPU core |
| `250m` | One-quarter of a CPU core |
| `100m` | One-tenth of a CPU core |

The suffix `m` means millicpu, or one-thousandth of a CPU unit.

For example, `750m` represents 0.75 CPU cores.

--------------------------------------------------------------------

## Understanding Memory Units

Kubernetes supports decimal and binary memory quantities.

Common examples:

| Unit | Meaning |
|---|---|
| `K` | 1,000 bytes |
| `M` | 1,000,000 bytes |
| `G` | 1,000,000,000 bytes |
| `Ki` | 1,024 bytes |
| `Mi` | 1,048,576 bytes |
| `Gi` | 1,073,741,824 bytes |

For example, `512Mi` means 512 mebibytes, while `512M` means 512 megabytes.

Use the correct units in manifests because decimal and binary quantities are not identical.

--------------------------------------------------------------------

## Quality of Service (QoS) Classes

Kubernetes assigns Pods to Quality of Service (QoS) classes based on their configured CPU and memory requests and limits.

The three classes are:

### Guaranteed

A Pod receives the `Guaranteed` QoS class when every container has CPU and memory requests and limits set, and each container's request equals its corresponding limit.

### Burstable

A Pod receives the `Burstable` class when it does not qualify as `Guaranteed` but has at least one CPU or memory request or limit set.

### BestEffort

A Pod receives the `BestEffort` class when no container in the Pod specifies CPU or memory requests or limits.

QoS classes can influence which Pods are selected first for eviction under node resource pressure. They do not replace appropriate resource planning or monitoring.

Check a Pod's QoS class with:

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
```

--------------------------------------------------------------------

## What Is Scaling?

Scaling means adjusting the capacity available to an application as its workload changes.

Kubernetes scaling can involve:

- Increasing or decreasing the number of Pod replicas.
- Adjusting CPU and memory resources assigned to workloads.
- Adding or removing nodes from a cluster.

These approaches solve different problems and may be combined.

--------------------------------------------------------------------

## Horizontal Pod Autoscaler (HPA)

The **Horizontal Pod Autoscaler (HPA)** automatically adjusts the number of replicas in a workload based on configured metrics.

For example, an HPA may increase the number of replicas when CPU utilization rises and reduce the count when demand falls.

An HPA commonly targets a Deployment or another scalable workload resource.

Example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

This example configures a target average CPU utilization of 70% relative to the relevant CPU requests. The HPA adjusts replicas between two and ten, subject to metric availability and scaling behaviour.

For CPU utilization-based autoscaling, containers need suitable CPU requests. The cluster also needs a functioning metrics source, commonly provided by Metrics Server for resource metrics.

Inspect HPA status:

```bash
kubectl get hpa
kubectl describe hpa nginx-hpa
```

**Important:** HPA does not automatically add more nodes to the cluster. It changes the desired replica count; node scaling is a separate concern.

--------------------------------------------------------------------

## Vertical Pod Autoscaler (VPA)

The **Vertical Pod Autoscaler (VPA)** helps recommend or adjust CPU and memory requests for containers based on observed resource usage.

Depending on its configuration and installed components, VPA can provide recommendations or apply resource changes. Some update modes may require Pods to be recreated to apply new resource settings.

VPA is not part of every Kubernetes installation by default. It generally requires separately installed components and configuration.

VPA is useful when the main problem is that a workload's CPU or memory requests need adjustment rather than that the application needs more replicas.

--------------------------------------------------------------------

## Cluster Autoscaler

The **Cluster Autoscaler** adjusts the number of nodes in a cluster when supported conditions are met.

For example:

- It may add nodes when Pods cannot be scheduled because available node capacity is insufficient and suitable node groups can grow.
- It may remove underutilized nodes when their workloads can be moved safely and the configured conditions permit removal.

Cluster Autoscaler requires compatible infrastructure integration and node-group configuration. It is not automatically available in every cluster.

**Remember:**

- HPA changes the number of Pod replicas.
- VPA adjusts or recommends per-container resource requests.
- Cluster Autoscaler changes the number of cluster nodes.

--------------------------------------------------------------------

## Metrics Server and Resource Monitoring

Metrics Server collects resource usage metrics from Kubernetes nodes and exposes them through the Metrics API.

It is commonly used by `kubectl top` and can provide resource metrics for HPA.

Check whether metrics are available:

```bash
kubectl top nodes
kubectl top pods
```

If these commands fail, verify that Metrics Server or another compatible metrics source is installed and working.

Metrics Server is not a complete long-term monitoring or logging solution. Production environments often use additional monitoring systems to collect historical metrics, dashboards, and alerts.

--------------------------------------------------------------------

## Common Resource Problems

### Pod Is Pending

Possible causes:

- The node does not have enough allocatable CPU or memory for the Pod's requests.
- Resource quotas or scheduling constraints prevent placement.
- No suitable node is available.

Inspect the Pod:

```bash
kubectl describe pod <pod-name>
kubectl get nodes
```

### Container Is OOMKilled

`OOMKilled` commonly indicates that a container was terminated because it exceeded a memory limit or the system encountered memory pressure.

Inspect the container status and recent events:

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

Review the memory request and limit, actual usage, and application memory behaviour before changing configuration.

### CPU Usage Is High

High CPU usage can result from increased traffic, expensive application operations, inefficient code, or inadequate CPU capacity.

Inspect available metrics:

```bash
kubectl top pods
kubectl top nodes
```

Use monitoring and application-level profiling when available to identify the underlying cause.

### HPA Does Not Scale

Possible causes:

- Metrics are unavailable.
- CPU requests are missing for containers involved in the utilization calculation.
- The HPA target or scale target is incorrect.
- The workload is already at its configured minimum or maximum.
- Scaling behaviour or cluster capacity limits further scaling.

Inspect the HPA and its events:

```bash
kubectl describe hpa <hpa-name>
kubectl get deployment
kubectl get pods
```

--------------------------------------------------------------------

## Key Takeaways

- Requests influence scheduling; limits constrain container resource usage.
- CPU limits can throttle a container, while memory-limit violations can result in termination.
- Kubernetes assigns each Pod a QoS class based on its CPU and memory requests and limits.
- HPA scales the number of replicas.
- VPA recommends or adjusts CPU and memory requests.
- Cluster Autoscaler adjusts node capacity when configured and supported.
- Metrics Server supplies resource metrics for commands and autoscaling.
- Resource settings should be based on observed usage and workload requirements.
- Scaling Pods and scaling cluster nodes are related but different operations.

--------------------------------------------------------------------

## Revision Questions

1. What is the difference between a resource request and a resource limit?
2. What does `500m` CPU represent?
3. What is the difference between `Mi` and `M`?
4. How are Guaranteed, Burstable, and BestEffort QoS classes determined?
5. What does the Horizontal Pod Autoscaler change?
6. What is the purpose of the Vertical Pod Autoscaler?
7. How does Cluster Autoscaler differ from HPA?
8. What is Metrics Server used for?
9. What can cause a container to be OOMKilled?
10. Which commands help diagnose resource usage and autoscaling problems?

--------------------------------------------------------------------