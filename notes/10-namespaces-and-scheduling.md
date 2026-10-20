# Namespaces and Scheduling

## What Are Namespaces?

A **Namespace** is a Kubernetes mechanism for organizing resources within a cluster. Namespaces help separate groups of resources, such as applications, teams, or environments.

For example, a cluster might use separate namespaces for development, testing, and production workloads.

Namespaces are useful for:

- Organizing Kubernetes resources.
- Separating resources between teams or environments.
- Applying namespace-scoped access controls.
- Applying resource quotas and default resource limits.
- Simplifying resource management in larger clusters.

Namespaces provide logical organization, not complete security isolation. Strong isolation may also require RBAC, network policies, appropriate workload configuration, and other security controls.

--------------------------------------------------------------------

## Common Namespaces

Kubernetes commonly includes these namespaces:

| Namespace | Purpose |
|---|---|
| `default` | Used when no namespace is specified for a namespaced resource. |
| `kube-system` | Contains many resources used by Kubernetes system components. |
| `kube-public` | A namespace readable by unauthenticated users by default in standard configurations; avoid assuming all clusters use identical permissions. |
| `kube-node-lease` | Contains Lease objects used for node heartbeats. |

List namespaces:

```bash
kubectl get namespaces
```

You can also use the shorter command:

```bash
kubectl get ns
```

--------------------------------------------------------------------

## Creating and Using Namespaces

Create a namespace:

```bash
kubectl create namespace development
```

List Pods in a specific namespace:

```bash
kubectl get pods -n development
```

Create a Deployment in a namespace:

```bash
kubectl create deployment nginx-demo --image=nginx:stable -n development
```

Inspect the Deployment:

```bash
kubectl get deployments -n development
```

The `-n` or `--namespace` flag specifies the namespace for a command.

Delete the namespace when it is no longer required:

```bash
kubectl delete namespace development
```

**Important:** Deleting a namespace also deletes the namespaced resources inside it. Check its contents before deleting it.

--------------------------------------------------------------------

## Namespaced and Cluster-Scoped Resources

Not every Kubernetes resource belongs to a namespace.

### Namespaced Resources

Examples include:

- Pods.
- Deployments.
- ReplicaSets.
- Services.
- ConfigMaps.
- Secrets.
- PersistentVolumeClaims.

These resources exist within a particular namespace.

### Cluster-Scoped Resources

Examples include:

- Nodes.
- Namespaces.
- PersistentVolumes.
- StorageClasses.
- ClusterRoles.
- ClusterRoleBindings.

These resources exist at the cluster level rather than inside an individual namespace.

Check where a resource is scoped:

```bash
kubectl api-resources
```

The `NAMESPACED` column indicates whether a resource is namespaced.

--------------------------------------------------------------------

## Working with Resources Across Namespaces

List Pods in the current namespace:

```bash
kubectl get pods
```

List Pods in a specific namespace:

```bash
kubectl get pods -n development
```

List Pods across all namespaces:

```bash
kubectl get pods --all-namespaces
```

The short form is:

```bash
kubectl get pods -A
```

View a Deployment in another namespace:

```bash
kubectl describe deployment nginx-demo -n development
```

The namespace must be specified correctly when working with resources outside the current namespace.

--------------------------------------------------------------------

## Setting a Default Namespace for a Context

A Kubernetes context stores connection information, including the cluster, user identity, and optionally a default namespace.

Inspect the active context:

```bash
kubectl config current-context
```

View available contexts:

```bash
kubectl config get-contexts
```

Set the default namespace for the current context:

```bash
kubectl config set-context --current --namespace=development
```

After this change, commands that do not specify `-n` use the configured default namespace for that context.

This changes the local kubeconfig context setting. It does not move existing resources between namespaces.

--------------------------------------------------------------------

## What Is Kubernetes Scheduling?

**Scheduling** is the process of selecting a suitable node on which to run a Pod.

The Kubernetes scheduler evaluates factors such as:

- Available node resources compared with Pod requests.
- Node selectors and affinity rules.
- Taints and tolerations.
- Pod affinity and anti-affinity.
- Scheduling constraints and policies.

After a node is selected, the kubelet on that node works with the container runtime to start the Pod's containers.

The scheduler selects a node; it does not itself run the containers.

--------------------------------------------------------------------

## Node Selectors

A **nodeSelector** constrains a Pod to nodes with specified labels.

For example, first inspect the node labels:

```bash
kubectl get nodes --show-labels
```

A Pod can specify a node selector like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-selector-demo
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-node-1
  containers:
    - name: nginx
      image: nginx:stable
```

In this example, the Pod can be scheduled only on a node with the matching hostname label.

Replace `worker-node-1` with a real label value from your cluster. If no node matches, the Pod may remain Pending.

Node selectors are simple and useful when workloads must run on nodes with particular labels.

--------------------------------------------------------------------

## Node Affinity

**Node affinity** provides more expressive rules for selecting nodes based on labels.

It supports required rules and preferred rules.

- **Required affinity:** A hard scheduling requirement. If no node satisfies it, the Pod cannot be scheduled.
- **Preferred affinity:** A scheduling preference. The scheduler tries to satisfy it but may select another node.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                  - worker-node-1
  containers:
    - name: nginx
      image: nginx:stable
```

This example requires a matching node label. Replace the sample hostname with a real value from your cluster.

The term `IgnoredDuringExecution` means that if a node's labels later change so the rule is no longer satisfied, Kubernetes does not automatically evict the already-running Pod solely because of that change.

--------------------------------------------------------------------

## Taints and Tolerations

**Taints** are applied to nodes to repel Pods that do not tolerate them.

**Tolerations** allow a Pod to be considered for scheduling onto a node with a matching taint. A toleration does not guarantee that the Pod will be scheduled there; other scheduling requirements still apply.

Common taint effects include:

- `NoSchedule`: New Pods that do not tolerate the taint are not scheduled onto the node.
- `PreferNoSchedule`: The scheduler tries to avoid placing non-tolerating Pods on the node, but this is a preference rather than a strict prohibition.
- `NoExecute`: Non-tolerating Pods may be evicted from the node, and new non-tolerating Pods are not scheduled there.

Inspect node taints:

```bash
kubectl describe node <node-name>
```

Example toleration:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "special-workloads"
    effect: "NoSchedule"
```

This toleration matches a taint with the specified key, value, and effect. It does not, by itself, force the Pod onto that node.

--------------------------------------------------------------------

## Resource Quotas

A **ResourceQuota** limits aggregate resource consumption or the number of selected object types within a namespace.

Example:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
```

This quota limits the namespace to ten Pods and sets aggregate limits for CPU and memory requests and limits.

A quota can prevent new resources from being created when the applicable quota would be exceeded. Workloads may need resource requests or limits configured for quota enforcement, depending on the quota dimensions being used.

Inspect quotas:

```bash
kubectl get resourcequota -n development
kubectl describe resourcequota development-quota -n development
```

--------------------------------------------------------------------

## LimitRanges

A **LimitRange** can define default resource requests and limits, as well as minimum and maximum resource values for supported resource types within a namespace.

Example:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-defaults
  namespace: development
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      default:
        cpu: "500m"
        memory: "256Mi"
```

In this example, the LimitRange supplies default requests and limits for containers when they do not specify those values and the defaults apply.

ResourceQuota and LimitRange serve different purposes:

- ResourceQuota limits aggregate consumption or object counts within a namespace.
- LimitRange defines defaults and per-resource constraints within a namespace.

--------------------------------------------------------------------

## Troubleshooting Scheduling Problems

### Pod Is Pending

Possible causes include:

- Insufficient node resources for the Pod's requests.
- A node selector or required affinity rule that matches no nodes.
- Untolerated node taints.
- Resource quota or other admission constraints.
- Persistent storage or other scheduling requirements that cannot be satisfied.

Inspect the Pod and its events:

```bash
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
```

Look for scheduler messages such as `Insufficient cpu`, `Insufficient memory`, or messages about node affinity and taints.

### Resources Are in the Wrong Namespace

List the resources in the relevant namespace:

```bash
kubectl get all -n development
```

Remember that `kubectl get all` does not show every Kubernetes resource type. Query specific resources such as Secrets, ConfigMaps, or PersistentVolumeClaims when needed.

### Quota Prevents Resource Creation

Inspect the namespace's quotas and recent events:

```bash
kubectl describe resourcequota -n development
kubectl get events -n development --sort-by=.metadata.creationTimestamp
```

Compare current usage with the quota's hard limits.

--------------------------------------------------------------------

## Key Takeaways

- Namespaces organize namespaced resources inside a cluster.
- Namespaces provide logical separation, not complete security isolation.
- Some resources, such as Nodes and PersistentVolumes, are cluster-scoped.
- A Kubernetes context can specify a default namespace.
- The scheduler selects a suitable node for each Pod.
- Node selectors and node affinity constrain node placement.
- Taints repel Pods unless they tolerate the taints.
- Tolerations permit scheduling but do not guarantee placement.
- ResourceQuota controls aggregate namespace consumption or object counts.
- LimitRange defines defaults and per-resource constraints.

--------------------------------------------------------------------

## Revision Questions

1. What is a Kubernetes Namespace?
2. What is the difference between namespaced and cluster-scoped resources?
3. How do you list Pods across all namespaces?
4. How can you set the default namespace for the current context?
5. What does the Kubernetes scheduler do?
6. How does node affinity differ from nodeSelector?
7. What is the difference between a taint and a toleration?
8. What are the three common taint effects?
9. How does ResourceQuota differ from LimitRange?
10. Which commands help troubleshoot a Pod that remains Pending?

--------------------------------------------------------------------