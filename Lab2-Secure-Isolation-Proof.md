# Lab 2: Secure Isolation and Multi-Tenancy Report

## Course Information

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 2 - Secure Isolation & Multi-Tenancy  
**Session:** Session A & B (Weeks 3–4)  
**Name:** Hafizi Hasdi  
**Date:** 16 August 2026

## Objective

Demonstrate compute, network and storage isolation using a local `kind` cluster with Calico, capture before/after network isolation evidence, verify resource quotas and RBAC-based secret isolation, and demonstrate data remanence and secure deletion techniques.

## Environment Summary

The local lab used the following components; evidence screenshots are referenced in the Evidence section.

| Component | Purpose | Evidence |
| --- | --- | --- |
| Docker | Run containers / kind | evidence/docker_version.png |
| kind | Local Kubernetes cluster | evidence/kind_cluster.png |
| kubectl | Kubernetes control plane client | evidence/kubectl_version.png |
| Calico (CNI) | NetworkPolicy enforcement | evidence/calico_ready.png |
| curlimages/curl, nginx, alpine | Test images used in probes and volume tests | evidence/images_used.png |

## Setup — Cluster Setup (Policy enforcement)

Create a `kind` cluster with the default CNI disabled and install Calico so NetworkPolicy is enforced.

```bash
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
```

![Kind cluster creation](Evidence/Screenshot%202026-08-12%20183301.png)


## Step 1 — Two Tenants & Applications

Model two customers as separate namespaces and deploy a simple web server for each.

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80
kubectl get pods,svc -n tenant-a
```

![Pods and Services](Evidence/Screenshot%202026-08-12%20191125.png)

Evidence: Pods and Services for `tenant-a` and `tenant-b` (image above)

## Step 2 — Observe Default-Open Risk (Probe)

By default, namespace boundaries do not restrict pod-to-pod networking. Capture the cluster IP of tenant-b's service and probe from a pod in tenant-a.

```bash
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'
```

![Before probe HTTP 200](Evidence/Screenshot%202026-08-16%20173408.png)

Evidence: Before probe result (HTTP 200) shown above

## Step 3 — Contain the Noisy Neighbour (ResourceQuota)

Limit tenant resource usage so a single tenant cannot exhaust shared nodes.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF

kubectl describe resourcequota tenant-a-quota -n tenant-a
```

![ResourceQuota output](Evidence/Screenshot%202026-08-16%20174553.png)

Evidence: ResourceQuota description shown above

## Step 4 — Default-Deny Network Isolation

Apply a default-deny ingress policy to `tenant-b` and re-run the same probe from Step 3 — expect timeout/failure.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'
```

![Calico rollout / NetworkPolicy enforcement](Evidence/Screenshot%202026-08-16%20201925.png)

Evidence: NetworkPolicy enforcement and calico-related output (image above). Re-run of the probe should now timeout/fail.

## Step 5 — Storage & Secret Isolation (RBAC)

Create per-tenant secrets and a namespaced service account; verify `can-i` checks to show tenant-a cannot read tenant-b's secret.

```bash
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B
kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a
SA=system:serviceaccount:tenant-a:app-a
kubectl auth can-i get secrets -n tenant-a --as=$SA  # expect: yes
kubectl auth can-i get secrets -n tenant-b --as=$SA  # expect: no
```

![Remanence and verification](Evidence/Screenshot%202026-08-16%20201625.png)

Evidence: Remanence scan and secure-wipe outputs shown above

## Step 6 — Data Remanence & Secure Deletion

Demonstrate that deleted files can persist inside volumes and show an overwrite (secure wipe) before delete.

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'

docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; echo wiped'
```

![Remanence and verification](Evidence/Screenshot%202026-08-16%20181025.png)

Evidence: Remanence scan and secure-wipe outputs shown above



## Short-Answer Questions

Q1 - Namespaces only organize resources logically, so they don't restrict network routing by default, so any pod can reach any other pod across namespaces. A breach or malicious tenant in one namespace can directly attack another tenant's workloads, risking cross-tenant data breaches on infrastructure.

Q2 - Default-deny means all traffic is blocked unless explicitly permitted which is the opposite of the default Kubernetes behavior. In Task 2, My NetworkPolicy implements this by selecting all pods in tenant-b and blocking all Ingress traffic, with no allow rules specified.


Q3 - containers share the host kernel (weaker isolation), VMs have their own kernel (stronger isolation). Add a VM boundary for untrusted or high-level sensitivity tenants where kernel-level compromise is a real risk.

Q4 - data remanence is when a data is still persist in the system even after deleting. cryptographic erasure the preferred cloud solution is because even if the data is recovered, the data is encrypted, and the key is destroyed.

Q5 - Task 1 = compute (namespaces separating tenant workloads), Task 3 = compute (resource quotas), Task 2 = network (the open-by-default connectivity), Task 4 = network (the deny policy), Task 5 = storage (secrets/RBAC).

## Verification Commands

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

## Environment Verification Checklist

| Check | Status |
| --- | --- |
| Tenants separated into namespaces | Completed |
| Default-deny NetworkPolicy applied | Completed |
| ResourceQuota created for tenant-a | Completed |
| Per-tenant secrets created | Completed |
| Remanence and wipe demonstrated | Completed |

## Cleanup & Next Steps

```bash
kind delete cluster --name ccse-lab2
docker volume rm ccse-vol
```

- Add actual screenshots into the `evidence/` folder and update paths if needed.
- I can inline images or generate a PDF report on request.

---
Generated from Lab 2 activities and short-answer responses.
