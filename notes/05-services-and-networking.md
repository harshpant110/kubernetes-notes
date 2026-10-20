# Services and Networking

## What Is Kubernetes Networking?

Kubernetes networking allows Pods and applications to communicate with one another and with external clients.

Pods can be created, deleted, or replaced during normal application operations. Their IP addresses can change, so applications should not rely on individual Pod IP addresses as permanent endpoints.

A **Service** provides a stable network endpoint for accessing a logical group of Pods.

Kubernetes networking supports:

- Communication between Pods.
- Communication between applications in different workloads.
- Access to applications through Services.
- Service discovery using DNS.
- Exposing applications inside or outside the cluster.

--------------------------------------------------------------------

## What Is a Service?

A **Service** is a Kubernetes resource that provides a stable way to access a set of backend Pods.

A Service commonly uses a label selector to identify the Pods that receive traffic.

For example, an Nginx Deployment may manage three Pods. A Service can select those Pods and provide a single endpoint for clients to access them.

```text
                 Client
                    |
                    v
              Nginx Service
                    |
          +---------+---------+
          |         |         |
          v         v         v
        Pod 1     Pod 2     Pod 3
```

The Service provides a stable endpoint even when individual Pods are replaced.

--------------------------------------------------------------------

## Why Do We Need Services?

Pod IP addresses are not permanent application endpoints. If a Pod is deleted and replaced, the replacement may receive a different IP address.

Without a stable access mechanism, clients would need to discover the current Pod addresses themselves.

A Service helps by providing:

- A stable virtual IP address for applicable Service types.
- A stable DNS name within the cluster.
- A way to direct traffic to matching backend Pods.
- A consistent endpoint as Pods are replaced or scaled.

**Important:** A Service does not create or manage application replicas. A Deployment manages the application Pods, while a Service provides network access to matching Pods.

--------------------------------------------------------------------

## How a Service Selects Pods

Services commonly use label selectors to identify their backend Pods.

For example, suppose Pods have this label:

```yaml
labels:
  app: nginx
```

A Service can select them using:

```yaml
selector:
  app: nginx
```

The selector must match the intended Pods' labels. If it matches no Pods, the Service will not have the expected backends.

A Service's backend endpoints are maintained by Kubernetes as the matching Pods change.

--------------------------------------------------------------------

## Types of Kubernetes Services

Kubernetes provides several Service types. The most common types to understand initially are `ClusterIP`, `NodePort`, and `LoadBalancer`.

### 1. ClusterIP

`ClusterIP` exposes a Service on an internal IP address accessible from within the cluster.

It is the default Service type.

Typical use cases:

- Internal APIs.
- Backend services.
- Communication between application components.
- Databases intended to be accessed internally.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

The Service is reachable from within the cluster using its Service IP or DNS name, subject to networking and policy configuration.

A `ClusterIP` Service is not directly exposed to external clients by default.

--------------------------------------------------------------------

### 2. NodePort

`NodePort` exposes a Service on a port across the cluster's nodes, subject to the cluster's network configuration.

External clients can access the application through a node's reachable IP address and the assigned NodePort.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

In this example:

- `port: 80` is the port exposed by the Service.
- `targetPort: 80` is the destination port on the backend Pods.
- `nodePort: 30080` is the port exposed on the nodes.

The default NodePort allocation range is commonly `30000–32767`, although cluster configuration can change it.

**Important:** A NodePort does not guarantee that the application is reachable from the public internet. Node addresses, routing, firewalls, and the cluster environment determine actual reachability.

--------------------------------------------------------------------

### 3. LoadBalancer

`LoadBalancer` requests an external load balancer through an appropriate cloud provider or supported infrastructure integration.

Typical use cases:

- Exposing a web application externally.
- Providing a public endpoint for an API.
- Receiving external traffic through a managed load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Kubernetes requests a load balancer, but an external address is not guaranteed to appear in every environment. The cluster must have a compatible implementation.

For example, local clusters may require additional configuration to provide load-balancer functionality.

--------------------------------------------------------------------

## Compare Service Types

| Service type | Main purpose | Typical access |
|---|---|---|
| `ClusterIP` | Internal application access | Inside the cluster |
| `NodePort` | Expose a Service through node ports | Through reachable node IPs and the NodePort |
| `LoadBalancer` | Request an external load balancer | Through the provisioned load-balancer endpoint |

Choose the Service type according to the application's access requirements and the capabilities of the cluster.

--------------------------------------------------------------------

## Understanding `port`, `targetPort`, and `nodePort`

These fields describe different parts of the traffic path.

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

| Field | Meaning |
|---|---|
| `port` | Port exposed by the Service. |
| `targetPort` | Port on the backend Pod to which traffic is directed. |
| `nodePort` | Port exposed on cluster nodes for a `NodePort` Service. |

In this example, traffic reaching the Service on port `80` is directed to the selected Pod's port `8080`. If the Service is of type `NodePort`, clients may also access it through the node's reachable IP and port `30080`.

The application must actually listen on the target port. Declaring `containerPort` in a Pod manifest does not, by itself, make the application listen on that port.

--------------------------------------------------------------------

## Service YAML Example

The following example defines a Service for Pods labelled `app: nginx`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### Explanation of the Fields

- `apiVersion: v1` identifies the core Kubernetes API version.
- `kind: Service` declares a Service resource.
- `metadata.name` gives the Service its name.
- `spec.type` defines how the Service is exposed.
- `spec.selector` identifies the backend Pods by label.
- `protocol: TCP` specifies the transport protocol.
- `port` defines the Service port.
- `targetPort` specifies the backend destination port.

**Note:** This example selects Pods labelled `app: nginx`. The selector must match your actual Pods' labels for the Service to route traffic to them.

--------------------------------------------------------------------

## Creating and Inspecting Services

### List Services

```bash
kubectl get services
```

The shorter form is:

```bash
kubectl get svc
```

### Inspect a Service

```bash
kubectl describe service nginx-service
```

### Display Service Details and IP Addresses

```bash
kubectl get svc -o wide
```

### List Pods and Their Labels

```bash
kubectl get pods --show-labels
```

### Inspect Service Endpoints

```bash
kubectl get endpoints nginx-service
```

On modern Kubernetes clusters, EndpointSlices are the preferred API for representing endpoint information. You can inspect them with:

```bash
kubectl get endpointslices
```

These commands help verify whether the Service exists, which selector it uses, which ports it exposes, and whether matching backend endpoints are available.

--------------------------------------------------------------------

## Creating a Service with `kubectl expose`

Kubernetes can create a Service for an existing workload using `kubectl expose`.

For example, to expose an existing Deployment internally:

```bash
kubectl expose deployment nginx-deployment \
  --name=nginx-service \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

This command creates a Service named `nginx-service` targeting the Pods selected through the Deployment.

Inspect the result:

```bash
kubectl get svc nginx-service
kubectl describe svc nginx-service
```

Before running this command, check whether a Service with the same name already exists in the namespace.

--------------------------------------------------------------------

## Service Discovery and DNS

Kubernetes provides DNS-based service discovery through a cluster DNS implementation.

Within the same namespace, a client can commonly access a Service using its name.

For example:

```text
http://nginx-service
```

A Service can also be addressed using a namespace-qualified name:

```text
nginx-service.default
```

The fully qualified Service DNS name commonly follows this pattern:

```text
<service-name>.<namespace>.svc.cluster.local
```

For example:

```text
nginx-service.default.svc.cluster.local
```

The cluster domain is configurable, so `cluster.local` is common but not guaranteed in every cluster.

DNS names simplify service-to-service communication because clients do not need to hard-code individual Pod IP addresses.

--------------------------------------------------------------------

## How Traffic Reaches a Service

A simplified traffic path for a ClusterIP Service is:

```text
Application Client
       |
       v
Service DNS Name
       |
       v
Service Virtual IP
       |
       v
Matching Backend Pod
```

Kubernetes networking components implement Service traffic routing. Depending on the cluster, this may involve `kube-proxy` or an alternative networking implementation.

A Service can route traffic to eligible backend Pods that match its selector. Readiness and endpoint configuration affect which Pods are used.

--------------------------------------------------------------------

## Internal and External Access

### Internal Access

A `ClusterIP` Service is suitable when an application should be accessed from inside the cluster.

For example, a frontend Pod can call an internal backend Service using its DNS name.

### External Access

A `NodePort` or `LoadBalancer` Service can provide external access, depending on the cluster's networking setup.

Other Kubernetes resources, such as an Ingress or Gateway implementation, can provide HTTP routing and related traffic-management capabilities when configured.

**Important:** Creating a Service alone does not guarantee external connectivity. Always verify the cluster environment, routing, firewall rules, and the actual endpoint.

--------------------------------------------------------------------

## Common Service Troubleshooting

If a Service cannot reach an application, inspect the Service, its selectors, and its backend endpoints.

```bash
# List Services
kubectl get svc

# Inspect the Service
kubectl describe svc nginx-service

# Check Pod labels
kubectl get pods --show-labels

# Check Pod readiness and IP addresses
kubectl get pods -o wide

# Inspect EndpointSlices
kubectl get endpointslices

# Inspect recent events
kubectl get events --sort-by=.metadata.creationTimestamp
```

Common causes include:

- The Service selector does not match the Pod labels.
- The Pods are not running or are not ready.
- The Service's `targetPort` does not match the application's listening port.
- The application is not listening on the expected interface or port.
- Network policies or cluster networking restrictions block traffic.
- A NodePort or LoadBalancer endpoint is not reachable from the client.
- The Service name or namespace is incorrect.

Troubleshoot systematically instead of assuming the Service itself is broken.

--------------------------------------------------------------------

## Key Takeaways

- A Service provides a stable way to access a logical group of Pods.
- Pod IP addresses may change when Pods are replaced.
- Services commonly select backend Pods using labels.
- `ClusterIP` is the default type and is used for internal access.
- `NodePort` exposes a Service through a port on cluster nodes.
- `LoadBalancer` requests an external load balancer through a supported integration.
- `port`, `targetPort`, and `nodePort` have different purposes.
- Kubernetes DNS enables applications to discover Services by name.
- Endpoint information helps determine whether a Service has matching backends.
- External reachability depends on cluster networking and infrastructure configuration.

--------------------------------------------------------------------

## Revision Questions

1. What problem does a Kubernetes Service solve?
2. Why should applications avoid relying on individual Pod IP addresses?
3. What is the difference between a Deployment and a Service?
4. What is the purpose of a Service selector?
5. What is the difference between `ClusterIP`, `NodePort`, and `LoadBalancer`?
6. Explain `port`, `targetPort`, and `nodePort`.
7. How can you list and inspect Services?
8. How does Kubernetes DNS help applications communicate?
9. What could cause a Service to have no backend endpoints?
10. Why does creating a `LoadBalancer` Service not guarantee public internet access?

--------------------------------------------------------------------