# Kubernetes Troubleshooting

## What Is Troubleshooting?

Troubleshooting is the process of identifying, diagnosing, and resolving problems in a Kubernetes cluster or application.

Kubernetes applications depend on several components working together, including:

- Pods and containers.
- Deployments and ReplicaSets.
- Services and networking.
- ConfigMaps and Secrets.
- Storage resources.
- Node resources and scheduling.
- Cluster components and infrastructure.

A problem in any of these areas can affect application availability.

The most effective troubleshooting approach is to inspect the current state, examine events and logs, identify the likely cause, and make a targeted change.

--------------------------------------------------------------------

## A Systematic Troubleshooting Workflow

Follow a consistent process instead of changing multiple settings at once.

1. **Identify the affected resource.** Determine whether the problem concerns a Pod, Deployment, Service, storage resource, or node.
2. **Check the current state.** Use `kubectl get` to inspect resource status.
3. **Inspect details and events.** Use `kubectl describe` to review conditions, configuration, and recent events.
4. **Check logs.** Use `kubectl logs` to investigate application errors.
5. **Check dependencies.** Verify configuration, networking, storage, and resource availability as appropriate.
6. **Apply a targeted fix.** Change the configuration or application only after identifying a plausible cause.
7. **Verify recovery.** Confirm that the resource is healthy and the application behaves as expected.

This process helps avoid random changes that can create additional problems.

--------------------------------------------------------------------

## Essential Troubleshooting Commands

### Inspect Resources

```bash
# List Pods in the current namespace
kubectl get pods

# List Pods across all namespaces
kubectl get pods -A

# List Deployments and Services
kubectl get deployments
kubectl get services

# Show node status
kubectl get nodes

# Display additional Pod and node information
kubectl get pods -o wide
kubectl get nodes -o wide
```

### Inspect a Resource

```bash
# Describe a Pod
kubectl describe pod <pod-name>

# Describe a Deployment
kubectl describe deployment <deployment-name>

# Describe a Service
kubectl describe service <service-name>

# Describe a node
kubectl describe node <node-name>
```

When the resource is in another namespace, specify it using `-n <namespace>`.

### Inspect Events

```bash
# Show events in the current namespace
kubectl get events --sort-by=.metadata.creationTimestamp

# Show events in a specific namespace
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Events often provide useful clues about scheduling failures, failed probes, image-pull errors, and resource constraints. Events may expire, so they are not a permanent audit history.

--------------------------------------------------------------------

## Pod Is Stuck in Pending

A Pod in `Pending` has not completed scheduling and startup. It may be waiting for a node, storage, or another requirement to be satisfied.

Common causes include:

- Insufficient CPU or memory for the Pod's resource requests.
- Node selectors or required affinity rules that match no nodes.
- Untolerated node taints.
- PersistentVolumeClaims that cannot be bound.
- Resource quota or admission constraints.
- No suitable node available.

Start with:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get nodes
kubectl get pvc
```

Look at the Pod's events for messages such as `Insufficient cpu`, `Insufficient memory`, or scheduling constraints that cannot be satisfied.

**Important:** A Pod can be `Pending` before its containers have started, so container logs may not yet be available.

--------------------------------------------------------------------

## Pod Is in ImagePullBackOff or ErrImagePull

These states indicate that Kubernetes cannot successfully retrieve the container image.

- `ErrImagePull` indicates an image-pull attempt failed.
- `ImagePullBackOff` indicates Kubernetes is delaying subsequent attempts after a pull failure.

Common causes include:

- An incorrect image name or tag.
- A missing image in the registry.
- Private registry authentication problems.
- Network or registry availability issues.
- Registry rate limits or access restrictions.

Inspect the Pod:

```bash
kubectl describe pod <pod-name>
```

Check the image reference in the workload manifest. For a private registry, verify that the required credentials and image-pull configuration are correct.

Do not assume that restarting the Pod will fix an incorrect image name or missing registry permission.

--------------------------------------------------------------------

## Container Is in CrashLoopBackOff

`CrashLoopBackOff` means a container is repeatedly terminating and Kubernetes is backing off before trying to restart it again.

Possible causes include:

- An application crash.
- Incorrect startup commands or arguments.
- Missing or invalid configuration.
- An unavailable dependency.
- A failed liveness or startup probe.
- Memory exhaustion or another runtime failure.

Inspect the Pod and its logs:

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

The `--previous` option retrieves logs from a previous terminated container instance when those logs are available.

If the Pod contains multiple containers, specify the container:

```bash
kubectl logs <pod-name> -c <container-name>
```

Review the container's exit code, restart count, events, and previous logs to narrow down the cause.

--------------------------------------------------------------------

## Pod Is Running but Not Ready

A container can be running without being ready to receive normal Service traffic.

Possible causes include:

- A failing readiness probe.
- Application initialization that has not completed.
- A dependency that is unavailable.
- An incorrect health-check path or port.
- An application that cannot currently serve requests.

Inspect the Pod:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

Review readiness-probe configuration and application logs. A readiness failure does not, by itself, restart the container.

--------------------------------------------------------------------

## Service Is Not Reachable

A Service provides a stable way to reach selected Pods, but connectivity depends on correct configuration and healthy endpoints.

Common causes include:

- Incorrect Service selectors.
- Pod labels that do not match the Service selectors.
- Pods that are not ready.
- Incorrect `port` or `targetPort` values.
- Network policies or other networking restrictions.
- An incorrect access method for the Service type.

Inspect the Service and its endpoints:

```bash
kubectl get services
kubectl describe service <service-name>
kubectl get endpoints <service-name>
kubectl get endpointslices
kubectl get pods --show-labels
```

EndpointSlices are the modern API used to represent Service endpoints. The legacy Endpoints resource may also be available, but EndpointSlices are generally preferred for detailed endpoint inspection.

Verify that the Service selector matches the intended Pod labels and that the application is listening on the expected port.

For a `ClusterIP` Service, the address is normally intended for access from within the cluster. External access may require a suitable Service type, Ingress, Gateway, or another configured network path.

--------------------------------------------------------------------

## Deployment Has Fewer Ready Replicas Than Expected

A Deployment attempts to maintain its desired replica count through its ReplicaSets.

Possible causes include:

- Pods failing to start.
- Image-pull errors.
- Insufficient node resources.
- Failed readiness or startup probes.
- Scheduling constraints.
- Application crashes.

Inspect the Deployment and its Pods:

```bash
kubectl get deployments
kubectl describe deployment <deployment-name>
kubectl get replicasets
kubectl get pods
```

Check rollout progress:

```bash
kubectl rollout status deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>
```

Do not assume that increasing the replica count will solve the problem. First determine why the existing replicas are not becoming ready.

--------------------------------------------------------------------

## Configuration Problems with ConfigMaps and Secrets

Applications may fail when required configuration is missing, incorrectly named, or supplied in an unexpected format.

Possible causes include:

- A referenced ConfigMap or Secret does not exist in the Pod's namespace.
- The expected key is missing.
- An environment variable is configured incorrectly.
- A mounted configuration file has an unexpected path or value.
- The application has not reloaded changed configuration.

Inspect the configuration resources:

```bash
kubectl get configmaps
kubectl describe configmap <configmap-name>
kubectl get secrets
```

Avoid printing Secret values into terminal logs or sharing them in screenshots. Check Secret metadata and key names without exposing sensitive data.

ConfigMap or Secret updates do not automatically cause every application to reload its configuration. Behaviour depends on how the resource is consumed and how the application handles changes.

--------------------------------------------------------------------

## Persistent Storage Problems

Storage issues can prevent a Pod from starting or cause an application to fail when accessing its data.

Possible causes include:

- A PersistentVolumeClaim remains unbound.
- No suitable PersistentVolume or StorageClass is available.
- Storage provisioning fails.
- The volume cannot be mounted.
- File permissions prevent access.
- The storage backend is unavailable.

Inspect storage resources:

```bash
kubectl get pvc
kubectl get pv
kubectl describe pvc <pvc-name>
kubectl describe pod <pod-name>
```

Check Pod events for volume attachment or mount errors. Do not delete a PersistentVolumeClaim or PersistentVolume as a troubleshooting shortcut without understanding the storage configuration and data-retention implications.

--------------------------------------------------------------------

## Resource Problems and OOMKilled

Resource constraints can prevent Pods from being scheduled or cause containers to perform poorly or terminate.

Common indicators include:

- `Insufficient cpu` or `Insufficient memory` in scheduling events.
- `OOMKilled` in a terminated container's status.
- High CPU usage or CPU throttling.
- Repeated restarts under memory pressure.

Inspect resource settings and usage:

```bash
kubectl describe pod <pod-name>
kubectl top pods
kubectl top nodes
```

The `kubectl top` commands require a functioning metrics source, commonly Metrics Server.

Review requests, limits, actual usage, and application behaviour before changing resource values.

--------------------------------------------------------------------

## Node Is NotReady

A node marked `NotReady` may be unable to run workloads reliably.

Possible causes include:

- Node network connectivity problems.
- Kubelet failures.
- Resource pressure.
- Container runtime problems.
- Underlying virtual machine or physical machine failures.

Inspect the node:

```bash
kubectl get nodes
kubectl describe node <node-name>
```

Review node conditions and events. Cluster-level access or infrastructure logs may be needed to diagnose problems that cannot be explained from Pod-level information alone.

--------------------------------------------------------------------

## Useful Logs and Debugging Commands

### Read Application Logs

```bash
# Read current container logs
kubectl logs <pod-name>

# Follow logs
kubectl logs -f <pod-name>

# Read logs from a previous terminated container instance
kubectl logs <pod-name> --previous

# Select a container in a multi-container Pod
kubectl logs <pod-name> -c <container-name>
```

### Execute a Command Inside a Container

```bash
kubectl exec -it <pod-name> -- sh
```

This opens a shell if the container includes a compatible shell and the user has permission to execute commands. Minimal images may not contain `sh` or other debugging tools.

For a Pod with multiple containers:

```bash
kubectl exec -it <pod-name> -c <container-name> -- sh
```

Use `kubectl exec` carefully, especially in production environments. Commands executed inside a container can change application state.

--------------------------------------------------------------------

## When to Use Rollback

A Deployment rollout may introduce a faulty application version or configuration.

Inspect rollout status and history:

```bash
kubectl rollout status deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>
```

If a rollout is faulty and a previous revision is suitable, you can roll back:

```bash
kubectl rollout undo deployment/<deployment-name>
```

After a rollback, verify Pod readiness, rollout status, application logs, and actual application behaviour. A successful rollback command does not by itself prove that the application is fully healthy.

--------------------------------------------------------------------

## Common Troubleshooting Mistakes

Avoid these practices:

- Restarting Pods without inspecting events or logs.
- Increasing replicas without identifying why existing Pods are failing.
- Changing several unrelated settings at once.
- Deleting storage resources without checking data-retention consequences.
- Exposing Secret values while collecting debugging information.
- Assuming `Running` means the application is ready.
- Assuming a successful rollout guarantees correct application behaviour.
- Treating temporary events as a permanent record of cluster activity.

A systematic investigation is usually safer and faster than trial-and-error changes.

--------------------------------------------------------------------

## Key Takeaways

- Start with resource status, then inspect details, events, and logs.
- `Pending` often indicates scheduling, storage, or admission constraints.
- `ImagePullBackOff` indicates repeated image retrieval failures.
- `CrashLoopBackOff` indicates repeated container termination with restart backoff.
- A running Pod may still be unready.
- Service connectivity depends on selectors, endpoints, ports, readiness, and networking.
- ConfigMaps, Secrets, and persistent storage can affect application startup.
- Resource pressure and node failures can affect workload availability.
- Rollbacks should be followed by verification.
- Protect sensitive configuration while troubleshooting.

--------------------------------------------------------------------

## Revision Questions

1. What is the recommended order for troubleshooting a Kubernetes problem?
2. What can cause a Pod to remain Pending?
3. What is the difference between `ErrImagePull` and `ImagePullBackOff`?
4. What does `CrashLoopBackOff` indicate?
5. How do you inspect logs from a previous terminated container instance?
6. What should you check when a Service cannot reach its Pods?
7. Why can a Pod be Running but not Ready?
8. How can you investigate a PersistentVolumeClaim that is not bound?
9. What does `OOMKilled` commonly indicate?
10. Why should you verify application behaviour after a Deployment rollback?

--------------------------------------------------------------------