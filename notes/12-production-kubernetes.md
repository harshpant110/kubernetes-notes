# Production Kubernetes

## What Is Production Kubernetes?

Production Kubernetes refers to running Kubernetes workloads in environments where application availability, security, performance, and data integrity are important.

A production cluster requires more than deploying an application successfully. It must also be monitored, secured, maintained, and prepared to recover from failures.

Important production concerns include:

- High availability and fault tolerance.
- Resource management and scaling.
- Security and access control.
- Monitoring, logging, and alerting.
- Reliable application deployments and rollbacks.
- Persistent data protection and recovery.
- Cluster upgrades and maintenance.
- Backup and disaster recovery.

--------------------------------------------------------------------

## High Availability

High availability (HA) means designing a system to remain accessible despite certain component failures.

Kubernetes supports highly available applications through mechanisms such as:

- Running multiple replicas of an application.
- Distributing Pods across suitable nodes.
- Using readiness probes to avoid sending normal Service traffic to unready Pods.
- Using PodDisruptionBudgets to limit certain voluntary disruptions.
- Operating a highly available control plane where required.

### Multiple Replicas

A Deployment can maintain multiple replicas of an application.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web-app
          image: nginx:1.28
          ports:
            - containerPort: 80
```

Three replicas can improve availability, but they do not guarantee it. If all replicas run on the same failed node, the application may still become unavailable.

### Distributing Workloads

Pod topology spread constraints and pod anti-affinity can help distribute replicas across nodes or failure domains.

Distribution reduces the risk that one node or zone failure will affect every replica. Actual resilience depends on cluster capacity, scheduling rules, and the infrastructure available.

--------------------------------------------------------------------

## Resource Requests and Limits

Resource requests and limits help control how workloads use CPU and memory.

- **Requests** describe the resources used by the scheduler when selecting a node.
- **Limits** define resource usage constraints enforced by Kubernetes.

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

CPU is generally throttled when a container exceeds its CPU limit. A container that exceeds its memory limit may be terminated with an out-of-memory condition.

Production workloads should use resource values informed by observed application behaviour and appropriate capacity planning.

Avoid setting arbitrary limits without understanding the application's workload characteristics.

--------------------------------------------------------------------

## Autoscaling

Autoscaling adjusts resources in response to changing demand.

Common Kubernetes scaling mechanisms include:

- **Horizontal Pod Autoscaler (HPA):** adjusts the number of Pod replicas based on configured metrics.
- **Vertical Pod Autoscaler (VPA):** recommends or adjusts container resource requests, depending on its configuration and operating mode.
- **Cluster autoscaling:** adjusts the number of cluster nodes when supported by the infrastructure and autoscaler configuration.

The HPA commonly uses CPU or memory utilization, but suitable metrics and resource requests must be configured. Other metrics can be used when the required metrics integration is available.

Autoscaling does not replace capacity planning. Scaling can fail when the cluster has insufficient resources, quota restrictions, or unsuitable scheduling constraints.

--------------------------------------------------------------------

## Security and Access Control

Production clusters should follow the principle of least privilege: each user, service account, and workload should receive only the permissions it needs.

### Role-Based Access Control (RBAC)

Kubernetes RBAC controls access to API resources through roles and bindings.

- **Role:** defines permissions within a namespace.
- **ClusterRole:** defines permissions that can apply across a cluster or be used in a namespace binding.
- **RoleBinding:** grants the permissions in a Role or ClusterRole within a namespace.
- **ClusterRoleBinding:** grants a ClusterRole's permissions across the cluster.

Avoid granting broad administrative access when narrower permissions are sufficient.

### Protecting Secrets

Secrets can contain passwords, tokens, certificates, or other sensitive configuration.

Recommended practices include:

- Restricting access with RBAC.
- Enabling encryption at rest for Secret data where supported and configured.
- Avoiding Secret values in source control, logs, and screenshots.
- Using an external secret-management system when appropriate.
- Rotating credentials according to security requirements.

Base64 encoding is not encryption. Secret access and storage must be protected through appropriate controls.

### Network Policies

NetworkPolicies can restrict network traffic between selected Pods and other network endpoints.

Their effectiveness depends on the cluster's network plugin supporting and enforcing NetworkPolicy rules. Production policies should be tested to ensure required application traffic remains available.

--------------------------------------------------------------------

## Container and Image Security

Container security begins before an image is deployed.

Useful practices include:

- Using trusted image sources.
- Keeping base images and dependencies updated.
- Scanning images for known vulnerabilities.
- Avoiding unnecessary packages and tools in production images.
- Running containers as non-root where feasible.
- Applying appropriate security contexts and Linux capabilities.
- Avoiding privileged containers unless there is a justified requirement.
- Using controlled image versions rather than relying on mutable tags such as `latest`.

Image scanning can identify known vulnerabilities, but it cannot guarantee that an image is completely secure.

--------------------------------------------------------------------

## Monitoring, Logging, and Alerting

Production systems need visibility into application health and cluster behaviour.

### Monitoring

Monitoring tracks metrics such as:

- CPU and memory utilization.
- Pod restarts and readiness.
- Request rate, latency, and error rate.
- Node health and resource pressure.
- Persistent storage capacity and performance.

Metrics can help identify trends and detect performance problems before they become outages.

### Logging

Logs provide details about application events and failures.

Production logging should support collecting, retaining, and searching logs without depending solely on the lifetime of an individual Pod.

Useful log sources include application logs, container runtime logs, and relevant Kubernetes component logs.

### Alerting

Alerts notify operators when important conditions require attention.

Examples include:

- An application has too few ready replicas.
- Error rates or latency exceed defined thresholds.
- A node becomes unavailable.
- Persistent storage approaches capacity.
- Repeated restarts or resource pressure affect workloads.

Alerts should be actionable and should avoid excessive noise.

--------------------------------------------------------------------

## Health Checks and Reliability

Kubernetes provides different probe types to assess container health.

- **Startup probe:** determines whether an application has completed startup.
- **Readiness probe:** determines whether a container is ready to receive normal Service traffic.
- **Liveness probe:** determines whether a container should be restarted.

Probe settings should reflect the application's actual startup and health behaviour.

An overly aggressive liveness probe can cause unnecessary restarts. A readiness probe should check whether the application can serve requests, rather than simply whether its process exists.

--------------------------------------------------------------------

## Reliable Deployments and Rollbacks

Production deployments should be controlled and verifiable.

Useful practices include:

- Reviewing changes before deployment.
- Using version-controlled manifests or deployment configuration.
- Testing changes in a suitable non-production environment.
- Monitoring rollout status.
- Verifying application behaviour after deployment.
- Maintaining a practical rollback procedure.

Check rollout progress:

```bash
kubectl rollout status deployment/<deployment-name>
```

Review deployment history:

```bash
kubectl rollout history deployment/<deployment-name>
```

Roll back when an appropriate previous revision is available:

```bash
kubectl rollout undo deployment/<deployment-name>
```

A successful rollout or rollback command does not guarantee that the application is functioning correctly. Verify readiness, logs, and actual application behaviour.

--------------------------------------------------------------------

## Persistent Data, Backups, and Disaster Recovery

Production applications may depend on persistent data that must survive Pod replacement or infrastructure failure.

Important practices include:

- Choosing appropriate persistent storage.
- Monitoring storage capacity and performance.
- Backing up application data and relevant cluster configuration.
- Testing restoration procedures.
- Defining recovery time and recovery point objectives where appropriate.
- Documenting disaster recovery procedures.

PersistentVolumes and PersistentVolumeClaims do not automatically provide backups or disaster recovery.

A backup strategy should account for application consistency, storage-system capabilities, and the recovery requirements of the workload.

--------------------------------------------------------------------

## Cluster Maintenance and Upgrades

Kubernetes clusters require ongoing maintenance to remain secure and supported.

Maintenance activities may include:

- Applying supported security updates.
- Upgrading Kubernetes components according to a tested plan.
- Checking version compatibility.
- Reviewing deprecated APIs and configuration.
- Draining nodes safely before maintenance when appropriate.
- Monitoring workload availability during changes.
- Maintaining a tested recovery plan.

Before upgrading, review the relevant version-specific documentation and compatibility requirements. Do not assume every component or add-on can be upgraded independently without consequences.

--------------------------------------------------------------------

## Production Readiness Checklist

Use this checklist as a starting point before deploying a workload into production.

- [ ] Appropriate replica count and failure-domain distribution.
- [ ] Resource requests and limits reviewed.
- [ ] Health probes configured and tested.
- [ ] Access controlled through appropriate RBAC permissions.
- [ ] Secrets protected and excluded from source control.
- [ ] Container images reviewed and scanned.
- [ ] Network access restricted where appropriate.
- [ ] Monitoring, logs, and actionable alerts available.
- [ ] Deployment and rollback procedures tested.
- [ ] Persistent data backup and restoration procedures verified.
- [ ] Cluster maintenance and upgrade plans documented.
- [ ] Recovery responsibilities and procedures understood.

This checklist should be adapted to the application's architecture, risk level, and operational requirements.

--------------------------------------------------------------------

## Common Production Mistakes

Avoid these practices:

- Treating multiple replicas as a complete high-availability strategy.
- Running every replica on the same failure-prone node.
- Deploying workloads without resource requests.
- Granting unnecessary cluster-wide permissions.
- Assuming Base64 encoding protects Secret values.
- Relying only on a container's `Running` state to assess application health.
- Deploying mutable image tags without version control.
- Assuming persistent storage automatically includes backups.
- Upgrading a cluster without checking compatibility.
- Skipping verification after deployments and rollbacks.

--------------------------------------------------------------------

## Key Takeaways

- Production Kubernetes requires reliability, security, observability, and operational planning.
- Multiple replicas help availability, but placement and infrastructure failures also matter.
- Resource requests and limits support scheduling and resource control.
- Autoscaling depends on suitable metrics, configuration, and available capacity.
- RBAC, Secret protection, network policies, and image security reduce risk.
- Monitoring, logging, and alerting help operators detect and investigate failures.
- Health probes support reliable traffic handling and container recovery.
- Backups must be created and restoration procedures tested separately.
- Maintenance and upgrades require planning and compatibility checks.
- Production readiness must be verified through testing and operational evidence.

--------------------------------------------------------------------

## Revision Questions

1. What does high availability mean in Kubernetes?
2. Why do multiple replicas not guarantee application availability?
3. What is the difference between resource requests and limits?
4. How do HPA and cluster autoscaling differ?
5. What are the roles of Role, ClusterRole, RoleBinding, and ClusterRoleBinding?
6. Why is Base64 encoding not sufficient to protect Secret data?
7. What role do monitoring, logging, and alerting play in production?
8. How do readiness, liveness, and startup probes differ?
9. Why should an application be verified after a successful rollout or rollback?
10. Why do PersistentVolumes and PersistentVolumeClaims not replace backups?
11. What should be considered before upgrading a Kubernetes cluster?
12. Name several items that belong in a production readiness checklist.

--------------------------------------------------------------------