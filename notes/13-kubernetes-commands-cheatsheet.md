# Kubernetes Commands Cheatsheet

## 1. Cluster and Context

```bash
# Display Kubernetes client and cluster versions
kubectl version

# Display cluster information
kubectl cluster-info

# List cluster nodes
kubectl get nodes

# Show detailed node information
kubectl get nodes -o wide

# List available contexts
kubectl config get-contexts

# Display the current context
kubectl config current-context

# Switch to a different context
kubectl config use-context <context-name>
```

--------------------------------------------------------------------

## 2. Basic Resource Operations

```bash
# List Pods in the current namespace
kubectl get pods

# List Pods across all namespaces
kubectl get pods -A

# List common Kubernetes resources
kubectl get all

# Display additional resource details
kubectl get pods -o wide

# Create or update resources from a manifest
kubectl apply -f <file.yaml>

# Delete resources defined in a manifest
kubectl delete -f <file.yaml>

# Describe a resource
kubectl describe pod <pod-name>

# View a resource as YAML
kubectl get pod <pod-name> -o yaml

# View a resource as JSON
kubectl get pod <pod-name> -o json
```

**Note:** `kubectl get all` does not display every resource type in a namespace.

--------------------------------------------------------------------

## 3. Pods

```bash
# List Pods
kubectl get pods

# List Pods in a specific namespace
kubectl get pods -n <namespace>

# Show Pod IP addresses and node placement
kubectl get pods -o wide

# Describe a Pod
kubectl describe pod <pod-name>

# View container logs
kubectl logs <pod-name>

# Follow logs continuously
kubectl logs -f <pod-name>

# View logs from a previous terminated container instance
kubectl logs <pod-name> --previous

# Select a container in a multi-container Pod
kubectl logs <pod-name> -c <container-name>

# Execute a command inside a container
kubectl exec -it <pod-name> -- sh

# Delete a Pod
kubectl delete pod <pod-name>
```

Deleting a Pod managed by a Deployment or another controller usually causes the controller to create a replacement.

--------------------------------------------------------------------

## 4. Deployments and ReplicaSets

```bash
# List Deployments
kubectl get deployments

# Describe a Deployment
kubectl describe deployment <deployment-name>

# List ReplicaSets
kubectl get replicasets

# Create or update a Deployment from a manifest
kubectl apply -f deployment.yaml

# Scale a Deployment
kubectl scale deployment <deployment-name> --replicas=3

# Update a container image
kubectl set image deployment/<deployment-name> <container-name>=<image>:<tag>

# Check rollout status
kubectl rollout status deployment/<deployment-name>

# View rollout history
kubectl rollout history deployment/<deployment-name>

# Undo the latest rollout
kubectl rollout undo deployment/<deployment-name>

# Restart a Deployment
kubectl rollout restart deployment/<deployment-name>
```

Verify application health after scaling, restarting, updating, or rolling back a Deployment.

--------------------------------------------------------------------

## 5. Services and Networking

```bash
# List Services
kubectl get services

# List Services in all namespaces
kubectl get services -A

# Describe a Service
kubectl describe service <service-name>

# List legacy Endpoints resources
kubectl get endpoints

# List EndpointSlices
kubectl get endpointslices

# Show Pod labels
kubectl get pods --show-labels

# Expose a Deployment through a Service
kubectl expose deployment <deployment-name> --port=80 --target-port=8080

# Forward a local port to a Pod
kubectl port-forward pod/<pod-name> 8080:80

# Forward a local port to a Service
kubectl port-forward service/<service-name> 8080:80
```

Port-forwarding examples assume that the selected Pod or Service can accept traffic on the specified destination port.

--------------------------------------------------------------------

## 6. ConfigMaps and Secrets

```bash
# List ConfigMaps
kubectl get configmaps

# Describe a ConfigMap
kubectl describe configmap <configmap-name>

# Create a ConfigMap from a literal value
kubectl create configmap <name> --from-literal=APP_MODE=production

# List Secrets
kubectl get secrets

# Describe a Secret without displaying its decoded values
kubectl describe secret <secret-name>

# Create a Secret from a literal value
kubectl create secret generic <name> --from-literal=PASSWORD=<value>
```

Avoid putting real passwords or tokens directly into shell commands, command history, or shared terminal output. Use an appropriate secure method for handling sensitive values.

--------------------------------------------------------------------

## 7. Storage

```bash
# List PersistentVolumeClaims
kubectl get pvc

# List PersistentVolumes
kubectl get pv

# List StorageClasses
kubectl get storageclass

# Describe a PersistentVolumeClaim
kubectl describe pvc <pvc-name>

# Describe a PersistentVolume
kubectl describe pv <pv-name>
```

A PersistentVolumeClaim in `Pending` may indicate that no suitable storage is available or that dynamic provisioning cannot complete.

Do not delete storage resources until you understand their data-retention and reclaim-policy implications.

--------------------------------------------------------------------

## 8. Health Checks and Troubleshooting

```bash
# Inspect Pod status
kubectl get pods

# Inspect detailed Pod conditions and events
kubectl describe pod <pod-name>

# List recent events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check node health
kubectl get nodes

# Inspect a node
kubectl describe node <node-name>

# Check Deployment rollout status
kubectl rollout status deployment/<deployment-name>

# View logs from the previous container instance
kubectl logs <pod-name> --previous

# Check resource usage
kubectl top pods
kubectl top nodes
```

The `kubectl top` commands require a working metrics source, commonly Metrics Server.

### Common Pod Statuses

| Status | Typical meaning |
|---|---|
| `Pending` | Waiting for scheduling or another startup requirement |
| `ErrImagePull` | An image-pull attempt failed |
| `ImagePullBackOff` | Image retrieval is failing and retries are delayed |
| `CrashLoopBackOff` | A container repeatedly terminates and restart attempts are backed off |
| `Running` | The Pod is assigned to a node and its containers have started; this does not guarantee readiness |
| `Succeeded` | The Pod's containers completed successfully |
| `Failed` | The Pod's containers terminated unsuccessfully |

Use `kubectl describe pod` and `kubectl logs` to investigate the underlying cause rather than relying on the status alone.

--------------------------------------------------------------------

## 9. Namespaces

```bash
# List namespaces
kubectl get namespaces

# Create a namespace
kubectl create namespace <namespace>

# List Pods in a namespace
kubectl get pods -n <namespace>

# Apply a manifest in a specific namespace
kubectl apply -f <file.yaml> -n <namespace>

# Describe a Deployment in a namespace
kubectl describe deployment <deployment-name> -n <namespace>

# Delete a namespace
kubectl delete namespace <namespace>
```

Deleting a namespace can delete the namespaced resources it contains. Confirm the impact before running the command.

--------------------------------------------------------------------

## 10. Resource Requests, Limits, and Scaling

```bash
# Inspect a Pod's configured resources
kubectl describe pod <pod-name>

# Inspect resource usage
kubectl top pods

# Inspect node resource usage
kubectl top nodes

# Scale a Deployment manually
kubectl scale deployment <deployment-name> --replicas=3

# List Horizontal Pod Autoscalers
kubectl get hpa

# Describe a Horizontal Pod Autoscaler
kubectl describe hpa <hpa-name>
```

Resource requests influence scheduling. Resource limits constrain container resource consumption. Autoscaling requires appropriate configuration, metrics, and available capacity.

--------------------------------------------------------------------

## 11. Scheduling and Node Management

```bash
# List nodes
kubectl get nodes -o wide

# Inspect node labels
kubectl get nodes --show-labels

# Describe a node
kubectl describe node <node-name>

# Add or update a node label
kubectl label node <node-name> <key>=<value>

# Remove a node label
kubectl label node <node-name> <key>-

# Mark a node unschedulable for new Pods
kubectl cordon <node-name>

# Drain a node before maintenance
kubectl drain <node-name> --ignore-daemonsets

# Allow new Pods to be scheduled on a node
kubectl uncordon <node-name>
```

Use `cordon` and `drain` carefully. Draining can be affected by PodDisruptionBudgets, local storage, DaemonSets, and workload configuration. Understand the impact before performing node maintenance.

--------------------------------------------------------------------

## 12. YAML Validation and Resource Inspection

```bash
# Validate a manifest through server-side dry run
kubectl apply --dry-run=server -f <file.yaml>

# Preview changes when supported by the resource and command
kubectl diff -f <file.yaml>

# Display a resource's YAML definition
kubectl get deployment <deployment-name> -o yaml

# Display a resource's JSON definition
kubectl get deployment <deployment-name> -o json

# Explain a resource field
kubectl explain pod.spec

# Explain Deployment fields
kubectl explain deployment.spec
```

A server-side dry run checks whether the API server accepts the proposed operation without persisting the change. It does not prove that the application will run correctly.

--------------------------------------------------------------------

## 13. Output and Filtering

```bash
# Display resources with additional details
kubectl get pods -o wide

# Display resource names only
kubectl get pods -o name

# Display selected fields using JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Watch resource changes
kubectl get pods -w

# Display resources across namespaces
kubectl get pods -A
```

JSONPath expressions are useful for extracting specific fields from Kubernetes API responses.

--------------------------------------------------------------------

## 14. Command Syntax Patterns

| Pattern | Purpose |
|---|---|
| `kubectl get <resource>` | List resources |
| `kubectl describe <resource> <name>` | Inspect details and events |
| `kubectl apply -f <file.yaml>` | Create or update resources from a manifest |
| `kubectl delete -f <file.yaml>` | Delete resources defined in a manifest |
| `kubectl logs <pod-name>` | Read container logs |
| `kubectl exec -it <pod-name> -- <command>` | Execute a command in a container |
| `kubectl get <resource> -n <namespace>` | Scope a query to a namespace |
| `kubectl get <resource> -A` | List namespaced resources across namespaces |
| `kubectl rollout status deployment/<name>` | Monitor a Deployment rollout |
| `kubectl rollout undo deployment/<name>` | Roll back a Deployment |

Replace placeholders such as `<pod-name>`, `<namespace>`, and `<file.yaml>` with actual values before running a command.

--------------------------------------------------------------------

## 15. Important Safety Practices

- Inspect a resource before changing or deleting it.
- Use the correct Kubernetes context and namespace.
- Avoid exposing Secret values in terminal output or logs.
- Review the effect of deleting namespaces, PVCs, or other persistent resources.
- Prefer declarative manifests and version control for repeatable changes.
- Verify rollout status and application behaviour after changes.
- Use dry-run or diff options where appropriate.
- Follow least-privilege access practices.

--------------------------------------------------------------------

## Key Takeaways

- `kubectl get` provides a quick view of resource status.
- `kubectl describe` helps reveal configuration, conditions, and events.
- `kubectl logs` helps diagnose application failures.
- `kubectl apply -f` is a common way to manage resources declaratively.
- `kubectl rollout` commands help inspect, monitor, restart, and roll back Deployments.
- Namespace flags help target the correct scope.
- `kubectl top` reports resource usage when a metrics source is available.
- Commands that affect storage, namespaces, or nodes require extra care.
- Always verify the result of an operation instead of assuming it succeeded.

--------------------------------------------------------------------

## Revision Questions

1. How do `kubectl get` and `kubectl describe` differ?
2. How can you inspect logs from a previous terminated container instance?
3. Which commands help diagnose a Pod stuck in `Pending`?
4. How do you scale a Deployment to three replicas?
5. How do you check whether a Deployment rollout has completed?
6. How can you inspect a Service's endpoints?
7. How do you list resources across all namespaces?
8. What is the difference between `kubectl cordon` and `kubectl drain`?
9. Why should Secret values not be exposed in terminal output?
10. What should you verify after applying a manifest or rolling back a Deployment?

--------------------------------------------------------------------