# Lab 1: Account Security and IAM Report

## Course Information

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 1 - Account Security and IAM  
**Session:** Session B (Week 2)  
**Name:** Student name  
**Date:** 05 August 2026  

## Objective

The objective of this lab is to complete Session B of Lab 1 using the LocalStack AWS-compatible environment and a local Kubernetes cluster created with `kind`. The focus is on demonstrating that access control is not just about identity creation, but also about enforcing authorization through RBAC.

By the end of this session, the report should show that:

- Docker and LocalStack are available for the local lab environment.
- A local Kubernetes cluster is created and verified.
- Separate namespaces are created for development and production.
- A least-privilege Role and RoleBinding are configured for a developer service account.
- `kubectl auth can-i` is used to prove that RBAC allows the permitted operation and blocks the unauthorized ones.

## Environment Summary

The local lab environment was verified in the Kali Linux VM used for this session.

| Component | Verified Version / Status | Evidence |
| --- | --- | --- |
| Docker | Running locally | Screenshot evidence in this report |
| LocalStack | Running and healthy on port `4566` | Screenshot evidence in this report |
| Kubernetes | `kind` cluster created successfully | Screenshot evidence in this report |
| `kubectl` | Configured and able to query the cluster | CLI output screenshots |
| RBAC | Role and RoleBinding enforced using Kubernetes authorization | CLI output screenshots |

## Step 1: One-Time Environment Setup

Before starting the Kubernetes RBAC tasks, the local environment must be prepared.

### 1.1 Verify Docker is available

The first command confirms that Docker is installed and working:

```bash
docker --version
```

Docker is required because LocalStack is started as a container locally, and `kind` also uses Docker to run the Kubernetes control plane and worker nodes.

### 1.2 Start LocalStack

The Lab 1 environment was restarted with the LocalStack container already available from the previous session:

```bash
docker start localstack
```

A health check was then performed to verify that LocalStack was ready:

```bash
curl http://localhost:4566/_localstack/health
```

The screenshot below shows the LocalStack container being verified before the Kubernetes RBAC work began.

![LocalStack health verification for Session B](Screenshot%202026-08-05%20112849.png)

## Step 2: Create the Local Kubernetes Cluster

A local Kubernetes cluster was created using `kind` to demonstrate authorization enforcement in a real control plane.

### 2.1 Create the cluster

```bash
kind create cluster --name ccse-lab1
```

### 2.2 Verify the cluster

```bash
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```

This proves that the Kubernetes cluster is running and available for RBAC testing.

![Create and verify the Kubernetes cluster](Screenshot%202026-08-05%20112957.png)

## Task 5: Separate Environments with Namespaces

Kubernetes namespaces are used to isolate workloads and policies, forming a first security boundary between environments.

### 5.1 Create the namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

This creates two logical environments where RBAC can be tested independently.

![Create dev and prod namespaces](Screenshot%202026-08-05%20113115.png)

## Task 6: Define a Role and Bind It (Least Privilege)

A service account is used to represent a developer identity inside the cluster. A Role defines what actions are allowed, and a RoleBinding links that Role to the service account.

### 6.1 Create the service account

```bash
kubectl create serviceaccount dev-user -n dev
```

### 6.2 Create a Role that allows only pod read operations

```bash
kubectl create role pod-reader -n dev \
  --verb=get,list,watch \
  --resource=pods
```

### 6.3 Bind the Role to the service account

```bash
kubectl create rolebinding dev-user-binding -n dev \
  --role=pod-reader \
  --serviceaccount=dev:dev-user
```

This demonstrates least privilege in Kubernetes: the developer identity is allowed to read pods in the `dev` namespace but is not granted permissions for deletes or for the `prod` namespace.

## Task 7: Test That Access Control Works

The most important part of the lab is verifying RBAC enforcement using `kubectl auth can-i`.

### 7.1 Define the service account identity

```bash
SA=system:serviceaccount:dev:dev-user
```

### 7.2 Test allowed read access in `dev`

```bash
kubectl auth can-i list pods -n dev --as=$SA
```

Expected result: `yes`

### 7.3 Test denied delete access in `dev`

```bash
kubectl auth can-i delete pods -n dev --as=$SA
```

Expected result: `no`

### 7.4 Test cross-namespace access blocking in `prod`

```bash
kubectl auth can-i list pods -n prod --as=$SA
```

Expected result: `no`

These three outputs prove that authentication identifies the service account, while authorization decides what the service account is actually allowed to do.

![RBAC can-i verification results](Screenshot%202026-08-05%20113248.png)

## Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Attaching permissions to groups is more scalable and auditable. Instead of managing permissions one user at a time, administrators can assign policies once to a group and every member inherits the same set of permissions. This reduces administrative overhead and makes access reviews easier.

### Q2. What is the difference between an IAM User and an IAM Role?

An IAM User is a long-lived identity that can be used directly by a person or application. An IAM Role is a temporary identity used to grant permissions to a principal for a limited period, typically through trust relationships. Roles are commonly preferred for automation and temporary access.

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

The Analyst account was granted only the read-only S3 policy, so it could not alter resources or security settings. If the account were compromised, the attacker would have a limited set of actions, which reduces the potential damage or blast radius compared with a full admin account.

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?

A Role defines the permissions that can be performed within a namespace, such as `get`, `list`, and `watch` on Pods. A RoleBinding links the Role to a user, group, or service account, which is the entity that receives those permissions.

### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?

The developer service account failed to access `prod` because the Role and RoleBinding were only scoped to the `dev` namespace. This demonstrates the security principle of least privilege and environment isolation: access is granted only where it is explicitly allowed, and the same identity is denied access outside that boundary.

## Verification Command

The following command was used to prove that the cluster RBAC binding is present and configured correctly:

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

Paste the command output here as evidence that the binding exists in the cluster.

## Environment Verification Checklist

| Check | Status |
| --- | --- |
| Docker is installed | Completed |
| LocalStack container is running | Completed |
| Kubernetes cluster created with `kind` | Completed |
| `kubectl cluster-info` returns a reachable cluster | Completed |
| `dev` and `prod` namespaces created | Completed |
| Role created for pod read access | Completed |
| RoleBinding created for `dev-user` service account | Completed |
| `kubectl auth can-i` shows `yes` for dev read access | Completed |
| `kubectl auth can-i` shows `no` for delete in `dev` | Completed |
| `kubectl auth can-i` shows `no` for access in `prod` | Completed |

## Conclusion

Session B of Lab 1 was completed successfully in the local environment. The lab demonstrated that identity alone is not enough; access must also be enforced. By creating a Kubernetes cluster, separating the `dev` and `prod` namespaces, assigning a least-privilege Role, and binding it to a service account, the lab showed how RBAC can allow a safe read-only operation while preventing unauthorized actions such as deleting pods or crossing into another namespace.

This Week 2 session reinforced the principle that security is strongest when permissions are scoped to the minimum required level and enforced by the platform itself.
