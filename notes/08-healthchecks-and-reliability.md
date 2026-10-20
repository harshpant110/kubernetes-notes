# Healthchecks and Reliability

## What Is Reliability in Kubernetes?

Reliability means keeping an application available and functioning correctly despite routine failures, restarts, or changes in workload conditions.

Kubernetes provides mechanisms that help detect unhealthy containers, control traffic to Pods, and maintain the desired number of replicas.

Important reliability features include:

- Liveness probes.
- Readiness probes.
- Startup probes.
- Deployment replica management.
- Restart policies.
- Rolling updates and rollbacks.
- Resource requests and limits.
- Monitoring and troubleshooting.

These features work together, but Kubernetes cannot guarantee application availability by itself. Reliability also depends on application design, cluster capacity, storage, networking, and operational practices.

--------------------------------------------------------------------

## What Are Health Probes?

A **probe** is a health check performed by the kubelet to determine whether a container meets a particular health condition.

Kubernetes supports three main types of probes:

1. **Liveness probe:** Determines whether a container should be restarted because it is considered unhealthy.
2. **Readiness probe:** Determines whether a container is ready to receive traffic.
3. **Startup probe:** Determines whether an application has completed its startup process.

Each probe has a different purpose. Choosing the correct probe prevents unnecessary restarts and helps ensure traffic reaches ready application instances.

--------------------------------------------------------------------

## 1. Liveness Probe

A **liveness probe** checks whether a container is alive and functioning according to the configured health check.

If the liveness probe fails enough times to exceed its failure threshold, Kubernetes restarts the affected container according to the Pod's restart policy.

Liveness probes are useful when an application can become stuck or deadlocked and cannot recover without a restart.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

### Important Fields

- `httpGet`: Performs an HTTP request to the specified path and port.
- `initialDelaySeconds`: Delays the first probe after the container starts.
- `periodSeconds`: Sets the interval between probes.
- `timeoutSeconds`: Sets how long a probe may take before timing out.
- `failureThreshold`: Sets how many consecutive failures are required before the probe is considered failed.

A failed liveness probe can trigger a container restart. Therefore, the health check should identify a condition that restarting the container can reasonably fix.

--------------------------------------------------------------------

## 2. Readiness Probe

A **readiness probe** determines whether a container is ready to serve requests.

If the readiness probe fails, Kubernetes marks the Pod as not ready. For a normal selector-based Service, the Pod is generally excluded from ready traffic endpoints until it becomes ready again.

Unlike a liveness probe, a readiness probe failure does not directly trigger a container restart.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

Readiness probes are useful when an application needs time to initialize, temporarily cannot handle requests, or depends on resources required to serve traffic.

For example, an application may be running but not yet ready because it is still loading data or initializing internal components.

--------------------------------------------------------------------

## 3. Startup Probe

A **startup probe** checks whether an application has completed startup.

It is useful for applications that take a long time to initialize. While the startup probe has not succeeded, Kubernetes does not run the configured liveness and readiness probes for that container.

Example:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 5
  failureThreshold: 30
```

With these settings, the probe runs every five seconds and allows up to 30 consecutive failures before the startup probe is considered failed. This provides an approximate startup allowance of 150 seconds, depending on probe timing and execution.

If the startup probe fails permanently, Kubernetes restarts the container according to the Pod's restart policy.

Once the startup probe succeeds, liveness and readiness checks can begin.

--------------------------------------------------------------------

## Comparing the Three Probes

| Probe | Main purpose | Typical consequence of failure |
|---|---|---|
| Liveness | Detect a container that should be restarted | Container restart after the configured failure threshold |
| Readiness | Determine whether a container can serve traffic | Pod becomes not ready; normal Service traffic is withheld |
| Startup | Determine whether an application has finished starting | Container restart if startup repeatedly fails |

**Remember:**

- Liveness answers: “Should this container be restarted?”
- Readiness answers: “Should this Pod receive normal Service traffic?”
- Startup answers: “Has this application finished starting?”

--------------------------------------------------------------------

## Probe Types

Kubernetes supports several probe mechanisms.

### HTTP Probe

Sends an HTTP request to a configured path and port.

```yaml
httpGet:
  path: /health
  port: 8080
```

This is appropriate when the application exposes an HTTP health endpoint.

### TCP Probe

Attempts to establish a TCP connection to the configured port.

```yaml
tcpSocket:
  port: 8080
```

A successful connection indicates that something is accepting TCP connections on that port. It does not necessarily prove that the application is functioning correctly.

### Exec Probe

Runs a command inside the container and evaluates its exit status.

```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

An exit status of zero indicates success.

Exec probes can be useful for local application checks, but frequent execution may add overhead and should be configured thoughtfully.

--------------------------------------------------------------------

## Complete Deployment Example with Health Probes

The following example shows how probes can be configured in a Deployment's container specification.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-health-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-health-demo
  template:
    metadata:
      labels:
        app: nginx-health-demo
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
          startupProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
            failureThreshold: 3
```

This manifest is an example for learning, not a replacement for testing the actual application's health endpoints.

The probes all check `/` on port 80. In a real application, select endpoints that correctly represent startup completion, process health, and readiness to serve traffic.

--------------------------------------------------------------------

## Restart Policies

A Pod's `restartPolicy` determines how the kubelet handles containers that terminate.

The available policies are:

- `Always`
- `OnFailure`
- `Never`

For Pods managed by a Deployment, the restart policy must be `Always`.

Examples of workload behaviour:

- A container in a Deployment Pod may be restarted after it exits or fails a liveness check.
- A failed Pod may be replaced by the workload controller when necessary to maintain the desired replica count.
- A completed Job uses a different workload model and commonly uses `OnFailure` or `Never`.

A container restart and a Pod replacement are not the same event. A container may restart inside the same Pod, while a replacement Pod is a new Pod resource.

--------------------------------------------------------------------

## Replicas and High Availability

Deployments maintain a desired replica count through their ReplicaSets.

For example:

```yaml
spec:
  replicas: 3
```

This declares a desired count of three Pods.

If one Pod becomes unavailable, the Deployment's controllers attempt to restore the desired state. However, replacement takes time, and the application may temporarily have fewer ready replicas.

Multiple replicas can improve availability, but only when the cluster has enough capacity and the replicas are not all affected by the same failure.

Consider:

- Running replicas on different nodes when appropriate.
- Using readiness probes to control traffic.
- Providing sufficient CPU and memory.
- Avoiding a single point of failure in the application architecture.
- Using suitable disruption and scheduling policies.

Three replicas on a single node do not protect the application from that node failing.

--------------------------------------------------------------------

## Rolling Updates and Reliability

Deployments support rolling updates, allowing application versions to be replaced gradually.

During a rollout, Kubernetes can create new Pods and terminate old ones according to the Deployment's update strategy.

Readiness probes help prevent Pods that are not ready from receiving normal Service traffic. However, they do not guarantee that every update will succeed.

Useful commands include:

```bash
# Monitor rollout progress
kubectl rollout status deployment/nginx-deployment

# View Deployment revisions
kubectl rollout history deployment/nginx-deployment

# Inspect Deployment status and conditions
kubectl describe deployment nginx-deployment
```

If an update is faulty, a rollback may restore a previous Pod template. A rollback should be followed by verification of the application's actual behaviour.

--------------------------------------------------------------------

## Resource Requests and Limits

Health and reliability are also affected by resource allocation.

### Requests

A resource request specifies the amount of CPU or memory Kubernetes uses when scheduling a Pod.

### Limits

A resource limit places a boundary on the resources a container may consume.

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Here:

- `100m` CPU means one-tenth of a CPU core.
- `128Mi` is the requested memory.
- The limits specify maximum configured CPU and memory consumption for the container.

A container exceeding its memory limit may be terminated with an out-of-memory condition. CPU limits can throttle CPU usage.

Appropriate requests and limits help avoid resource contention, but values should be based on measured application requirements.

--------------------------------------------------------------------

## Monitoring Pod and Container Health

Use these commands to inspect application health:

```bash
# List Pods and their readiness status
kubectl get pods

# Show additional Pod details
kubectl get pods -o wide

# Inspect a Pod's conditions, probes, and events
kubectl describe pod <pod-name>

# Read container logs
kubectl logs <pod-name>

# Follow logs while the container runs
kubectl logs -f <pod-name>

# Inspect Deployment replica status
kubectl get deployments

# Inspect recent cluster events
kubectl get events --sort-by=.metadata.creationTimestamp
```

For a Pod with multiple containers, specify the container name when necessary:

```bash
kubectl logs <pod-name> -c <container-name>
```

These commands help distinguish a container that is running from a Pod that is ready to serve traffic.

--------------------------------------------------------------------

## Common Healthcheck Problems

### Liveness Probe Keeps Restarting the Container

Possible causes:

- The health endpoint is incorrect.
- The application takes longer to initialize than expected.
- Probe timeouts are too short.
- The application is genuinely stuck or unhealthy.

Review the probe configuration and container events before increasing thresholds.

### Pod Is Running but Not Ready

Possible causes:

- The readiness endpoint is failing.
- The application is still initializing.
- A required dependency is unavailable.
- The configured port or path is incorrect.

Inspect the Pod's conditions and events, and verify the application endpoint.

### Startup Probe Fails

Possible causes:

- The application takes longer to start than the configured allowance.
- The health endpoint is unavailable during startup.
- The probe uses the wrong path or port.
- The application fails to initialize correctly.

Configure the startup probe based on realistic startup times and actual application behaviour.

### Pods Are Repeatedly Terminated

Possible causes include:

- Liveness probe failures.
- Memory exhaustion.
- Application crashes.
- Node or infrastructure problems.

Inspect container status, restart counts, events, and previous logs when available.

--------------------------------------------------------------------

## Key Takeaways

- Liveness, readiness, and startup probes have different purposes.
- Liveness probes can trigger container restarts.
- Readiness probes control whether Pods are considered ready for normal Service traffic.
- Startup probes allow applications time to initialize.
- HTTP, TCP, and exec probes provide different health-check mechanisms.
- Deployment controllers help maintain the desired replica count.
- Multiple replicas do not automatically protect against every failure.
- Resource requests and limits influence scheduling and container behaviour.
- `kubectl describe pod`, logs, and events are essential troubleshooting tools.
- Health probes should represent real application conditions, not merely whether a process exists.

--------------------------------------------------------------------

## Revision Questions

1. What is the difference between liveness, readiness, and startup probes?
2. What happens when a liveness probe repeatedly fails?
3. Does a readiness probe failure automatically restart a container?
4. Why is a startup probe useful for slow-starting applications?
5. What are the HTTP, TCP, and exec probe types?
6. What is the difference between restarting a container and replacing a Pod?
7. How do multiple replicas improve availability, and what are their limitations?
8. What is the difference between resource requests and limits?
9. Which commands help diagnose a Pod that is not ready?
10. Why should health probes use meaningful application health checks?

--------------------------------------------------------------------