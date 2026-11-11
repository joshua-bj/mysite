---
publish: true
created: 2026-11-11T10:05:46.309Z
modified: 2026-11-11T10:10:35.930Z
---

## Why Persistent Storage?

A container's file system is ephemeral by design — anything written inside it disappears when the container is recreated. To persist data or access an external file system, you must use a **PersistentVolume (PV)**.

## Understanding Access Modes

A PersistentVolume can be exposed with different access modes:

| Access Mode | Abbreviation | Description |
| --- | --- | --- |
| `ReadWriteOnce` | RWO | The volume can be mounted as read-write by a **single node**. |
| `ReadOnlyMany` | ROX | The volume can be mounted as read-only by **many nodes**. |
| `ReadWriteMany` | RWX | The volume can be mounted as read-write by **many nodes**. |

Support for these modes varies by volume implementation, but as a general rule, the breadth of support follows this order:

> **RWO > ROX > RWX**

In other words, RWO is by far the most widely supported mode across cloud providers and storage backends.

> **References**
>
> - OpenShift: [Understanding persistent storage — Access modes](https://docs.openshift.com/container-platform/4.7/storage/understanding-persistent-storage.html#pv-access-modes_understanding-persistent-storage)
> - Kubernetes: [Persistent Volumes — Access Modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)

## Design Principles for Cloud-Native Applications

Given the reality above, two principles should guide your storage design:

1. **Use RWO volumes for application state.** This is data that only the application itself needs to access — no human ever needs to know or care where it physically resides. After a stateful pod recovers from a failure, it simply re-attaches to the external storage and finds its data waiting. Typical examples include an application engine's transaction logs or JMS messages.

2. **Use services, not file systems, for business data input and output.** In the cloud world, business data should flow through services such as Kafka or a content service rather than through shared file systems.

## Recommended Usage Patterns (CloudPak)

The following is the typical pattern used in CloudPak deployments — and in my view, the best practice.

### 1. Store configuration files in ConfigMaps and Secrets

This includes keystores and truststores.

- **Read-only inside the container.** Customers modify the content externally, and updates are projected into the running container almost immediately.
  - ⚠️ **Never use `subPath`** — it breaks the live update propagation.
- **Size limit:** the content cannot exceed **1 MiB**.

### 2. Use RWO volumes for application state

- The application **must be managed by a StatefulSet**. As the name suggests, you should not use a Deployment to manage a stateful application.
- Choose a storage provider that supports **dynamic provisioning** — this becomes essential when scaling the application up.

### 3. Use external services for business data input and output

Data in this category is _not_ application state, so you should not care which pod is processing — or has processed — the data.

> In theory, you could use an RWO volume for input and output data, but because it cannot be shared across cluster nodes, it becomes very difficult for administrators to manage. For this reason, I would not recommend RWO volumes for this pattern.

### 4. Never mix configuration storage with application data

For example, do not share a single RW volume between an application's input/output and its configuration.

The most obvious reason: **configuration and application data are owned by different roles**, and for security reasons, they should not be shared with each other.

## Summary

| Data type | Where it belongs | Key consideration |
| --- | --- | --- |
| Configuration files (incl. keystores/truststores) | ConfigMap / Secret | Read-only, live updates (no `subPath`), 1 MiB limit |
| Application state (transaction logs, JMS messages) | RWO volume via StatefulSet | Dynamic provisioning for scale-up |
| Business data input/output | External services (Kafka, content service) | Decoupled from any specific pod |
| Mixed config + data on one volume | ❌ Nowhere | Different owners; a security risk |
