# Kubernetes Notes

A structured, hands-on learning resource covering Kubernetes fundamentals, architecture, workload management, networking, storage, troubleshooting, and production concepts.

These notes document my journey learning Kubernetes through theory, practical exercises, YAML manifests, command-line operations, and troubleshooting in a local Kubernetes cluster.

--------------------------------------------------------------------

## Table of Contents

### Fundamentals

1. [Kubernetes Basics](notes/01-kubernetes-basics.md)
2. [Kubernetes Architecture](notes/02-kubernetes-architecture.md)
3. [Pods](notes/03-pods.md)

### Workloads and Networking

4. [Deployments and ReplicaSets](notes/04-deployments-and-replicasets.md)
5. [Services and Networking](notes/05-services-and-networking.md)

### Configuration and Storage

6. [ConfigMaps and Secrets](notes/06-configmaps-and-secrets.md)
7. [Storage and Persistent Volumes](notes/07-storage-and-persistent-volumes.md)

### Reliability and Resource Management

8. [Health Checks and Reliability](notes/08-healthchecks-and-reliability.md)
9. [Resources and Scaling](notes/09-resources-and-scaling.md)
10. [Namespaces and Scheduling](notes/10-namespaces-and-scheduling.md)

### Operations and Reference

11. [Troubleshooting](notes/11-troubleshooting.md)
12. [Production Kubernetes](notes/12-production-kubernetes.md)
13. [Kubernetes Commands Cheatsheet](notes/13-kubernetes-commands-cheatsheet.md)

--------------------------------------------------------------------

## Topics Covered

- Kubernetes fundamentals and cluster architecture
- Control plane and worker node components
- Pods, Deployments, and ReplicaSets
- Services, service discovery, and Ingress
- ConfigMaps and Secrets
- Persistent Volumes, Persistent Volume Claims, and StorageClasses
- Readiness, liveness, and startup probes
- Resource requests, limits, and scaling
- Namespaces, scheduling, and node selectors
- Troubleshooting Pod failures and application issues
- Rolling updates, rollbacks, security, and production reliability
- Essential `kubectl` commands

--------------------------------------------------------------------

## Practical Environment

The concepts and commands in these notes were practised using:

- **Operating System:** Ubuntu Linux
- **Container Runtime and Tooling:** Docker
- **Local Kubernetes Environment:** k3d with k3s
- **Command-Line Tool:** kubectl
- **Configuration Format:** YAML
- **Documentation Format:** Markdown

The practical exercises focus on deploying workloads, inspecting cluster resources, testing connectivity, configuring persistent storage, and diagnosing failures.

--------------------------------------------------------------------

## Learning Approach

Each topic is organized around:

- **Concepts:** What the resource or feature is and why it is used.
- **Configuration:** YAML examples and important fields.
- **Commands:** Practical `kubectl` commands.
- **Hands-on Practice:** Applying concepts in a local cluster.
- **Troubleshooting:** Understanding errors and verifying fixes.
- **Revision:** Key takeaways and interview-relevant concepts.

--------------------------------------------------------------------

## Project Application

The knowledge developed through these notes will be applied to my **JobFlow Distributed Job Processing Platform** as a separate practical phase.

The goal is to use Kubernetes to deploy and manage the application's components, configure networking and storage, handle application configuration, and validate the overall deployment.

--------------------------------------------------------------------

## Purpose

This repository serves as my Kubernetes learning reference for practical revision, technical interview preparation, and future DevOps projects.

**Learning Kubernetes by understanding the concepts, practising the commands, and troubleshooting real workloads.** 