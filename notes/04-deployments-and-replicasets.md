# Deployments and ReplicaSets

## What Is a Deployment?

A **Deployment** is a Kubernetes resource used to manage replicated, long-running application workloads.

It manages ReplicaSets, which maintain the desired number of Pod replicas. Deployments also support declarative updates and rollbacks of application versions.

A Deployment helps with:

- Maintaining the desired number of replicas.
- Replacing Pods when they are deleted or become unavailable.
- Scaling an application.
- Performing rolling updates.
- Rolling back to an earlier Deployment revision.

For most stateless applications, a Deployment is preferred over creating individual Pods manually.

--------------------------------------------------------------------

## What Is a ReplicaSet?

A **ReplicaSet** ensures that the specified number of matching Pods are maintained.

For example, if a ReplicaSet is configured for three replicas, its controller works to maintain three Pods matching its selector.

If a managed Pod is deleted, the ReplicaSet can create a replacement. The replacement may have a different name and IP address.

ReplicaSets are commonly managed by Deployments rather than created directly.

--------------------------------------------------------------------

## Relationship Between Deployments, ReplicaSets, and Pods

```text
                 Deployment
                      |
                 ReplicaSet
                      |
            +---------+---------+
            |         |         |
           Pod       Pod       Pod
         Replica 1 Replica 2 Replica 3
```

Responsibilities:

- **Deployment:** Manages rollout history, updates, rollbacks, and ReplicaSets.
- **ReplicaSet:** Maintains the desired number of matching Pods.
- **Pod:** Runs the application's containers.

During a rolling update, a Deployment may create a new ReplicaSet and gradually replace Pods from the previous ReplicaSet.

--------------------------------------------------------------------

## Creating a Deployment with YAML

The following manifest illustrates the Nginx Deployment used during the practical exercises.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

### Explanation of Important Fields

| Field | Purpose |
|---|---|
| `apiVersion: apps/v1` | Identifies the API version for Deployments. |
| `kind: Deployment` | Declares the resource type. |
| `metadata.name` | Names the Deployment. |
| `spec.replicas` | Declares the desired number of Pods. |
| `spec.selector.matchLabels` | Identifies the Pods managed by the Deployment's ReplicaSets. |
| `spec.template` | Defines the Pod template used to create replicas. |
| `template.metadata.labels` | Supplies the labels that must match the Deployment selector. |
| `containers` | Defines the containers for each Pod. |
| `readinessProbe` | Checks whether the container is ready to serve traffic. |

**Important:** The selector must match the labels in the Pod template. For `apps/v1` Deployments, the selector is immutable after creation.

The readiness probe checks the Nginx HTTP endpoint at `/`. A Pod that has not passed its readiness probe is not considered ready to serve traffic through a normal Service.

--------------------------------------------------------------------

## Creating and Inspecting a Deployment

### Apply a Manifest

```bash
kubectl apply -f deployment.yaml
```

### List Deployments

```bash
kubectl get deployments
```

### Inspect a Deployment

```bash
kubectl describe deployment nginx-deployment
```

### List ReplicaSets

```bash
kubectl get replicasets
```

### List Pods with Labels and Node Details

```bash
kubectl get pods -o wide --show-labels
```

### Check Rollout Status

```bash
kubectl rollout status deployment/nginx-deployment
```

These commands help verify the desired replica count, available replicas, current image, rollout progress, and the Pods managed by the Deployment.

--------------------------------------------------------------------

## Scaling a Deployment

Scaling changes the desired number of Pod replicas.

### Scale to Five Replicas

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Verify the result:

```bash
kubectl get deployment nginx-deployment
kubectl get pods -o wide
```

### Scale Back to Three Replicas

```bash
kubectl scale deployment nginx-deployment --replicas=3
```

Verify again:

```bash
kubectl get deployment nginx-deployment
kubectl get pods
```

During the practical exercise, the Deployment was scaled from three replicas to five and then returned to three. New Pods took time to start, so the Deployment did not necessarily reach the requested count immediately.

**Important:** Scaling changes the desired replica count; it does not necessarily change the application version.

If a Deployment manifest declares `replicas: 3`, applying that manifest after manually scaling to five can restore the declared count to three.

--------------------------------------------------------------------

## Rolling Updates

A rolling update gradually replaces old Pods with new Pods when the Deployment's Pod template changes, such as when its container image changes.

This helps update an application without intentionally stopping all replicas at once.

### Update the Container Image

For example:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

The container name `nginx` must match the name in the Deployment's Pod template.

This command changes the Deployment's image configuration. Use an image version appropriate for the application and verify that it is available before updating a real workload.

### Monitor the Rollout

```bash
kubectl rollout status deployment/nginx-deployment
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

Kubernetes creates a new ReplicaSet for the updated Pod template and gradually shifts replicas from the old version to the new one.

**Note:** This example is an update command for reference. Do not run it on your learning Deployment unless you intend to change the image and practise a rollout.

--------------------------------------------------------------------

## Rolling Update Strategy

Deployments commonly use the `RollingUpdate` strategy.

Two important settings are:

- **`maxUnavailable`:** The maximum number or proportion of desired Pods that may be unavailable during an update.
- **`maxSurge`:** The maximum number or proportion of additional Pods that may be created above the desired replica count during an update.

In your learning Deployment, the observed strategy was:

```text
StrategyType: RollingUpdate
MaxUnavailable: 25%
MaxSurge: 25%
```

With three desired replicas:

- `maxUnavailable: 25%` rounds down to zero Pods.
- `maxSurge: 25%` rounds up to one additional Pod.

This allows the rollout to create one extra Pod while keeping the desired replicas available, subject to readiness, capacity, and other cluster constraints.

A rollout can stall if new Pods cannot start or become ready.

--------------------------------------------------------------------

## Rollout History and Rollback

Kubernetes records Deployment revisions when changes to the Pod template trigger new revisions.

### View Rollout History

```bash
kubectl rollout history deployment/nginx-deployment
```

### Inspect a Specific Revision

```bash
kubectl rollout history deployment/nginx-deployment --revision=4
```

Replace `4` with the revision you want to inspect.

### Roll Back to the Previous Revision

```bash
kubectl rollout undo deployment/nginx-deployment
```

### Roll Back to a Specific Revision

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=4
```

Use these commands carefully: a rollback changes the Deployment's Pod template and may trigger another rollout.

The `CHANGE-CAUSE` field can display `<none>` if no cause was recorded. Revision history is useful, but it is not a complete audit log of every change.

--------------------------------------------------------------------

## Image Tags and Reproducibility

A container image reference identifies the image Kubernetes should run.

Examples:

```yaml
image: nginx:stable
```

```yaml
image: nginx:1.28
```

A mutable tag, such as `stable`, can point to a different image over time. A specific version tag is generally more predictable, but tags are not guaranteed to be immutable.

For stronger reproducibility, production workflows can pin images by digest.

Always verify the selected image and use a deliberate update and rollback process.

--------------------------------------------------------------------

## Deployment Troubleshooting

If a Deployment does not reach its desired state, inspect the Deployment, ReplicaSets, Pods, and events.

```bash
# Check Deployment status
kubectl get deployments

# Inspect Deployment conditions and events
kubectl describe deployment nginx-deployment

# Inspect ReplicaSets
kubectl get replicasets

# Inspect Pods
kubectl get pods -o wide

# Inspect a specific Pod
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Check rollout progress
kubectl rollout status deployment/nginx-deployment

# Inspect recent events
kubectl get events --sort-by=.metadata.creationTimestamp
```

Common causes include:

- An unavailable or incorrectly specified image.
- A mismatch between selectors and Pod labels.
- Insufficient cluster resources.
- Scheduling constraints.
- Failed readiness probes.
- Invalid application configuration.

Diagnose the actual cause before changing the replica count or rolling back.

--------------------------------------------------------------------

## Key Takeaways

- A Deployment manages ReplicaSets and supports application rollouts.
- A ReplicaSet maintains the desired number of matching Pods.
- The Pod template defines the Pods created by a Deployment.
- The Deployment selector must match the Pod template labels.
- Scaling changes the desired replica count.
- Rolling updates gradually replace Pods when the Pod template changes.
- `maxUnavailable` and `maxSurge` control aspects of rolling-update behaviour.
- Rollout history and rollback help manage application versions.
- Image tags are not always immutable; digests provide stronger image identity.
- A successful Deployment update still requires verification of Pod readiness and application behaviour.

--------------------------------------------------------------------

## Revision Questions

1. What is the difference between a Deployment and a ReplicaSet?
2. Why is a Deployment generally preferred over standalone Pods?
3. What is the purpose of `spec.replicas`?
4. Why must the selector match the Pod template labels?
5. What happens when a managed Pod is deleted?
6. How do you scale a Deployment from three replicas to five?
7. What triggers a rolling update?
8. What do `maxUnavailable` and `maxSurge` control?
9. How do you inspect rollout history and roll back a Deployment?
10. Why is a mutable image tag less predictable than an image digest?
