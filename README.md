# Kuberenetes Network Policy Debugging: Weave Net & Kube-Proxy SNAT Issue

## The Problem

A NetworkPolicy was created to restrict traffic to the `mysql` database pod so that only the `frontend` is restricted, meaning the `backend` pod should be the only component allowed to communicate with it on port 3306. 

However, despite the labels in the Network Policy logically matching, even the `backend` pod was timing out when attempting to connect to the DB Service layer (`telnet db 3306`).

## Context
- **CNI**: Weave Net
- **Target Pod**: `mysql` (Labeled `name=mysql`)
- **Allowed Source**: `backend` (Labeled `role=backend`)
- **Network Policy**: `db-test` (Matches `name=mysql`, explicitly allows ingress from `role=backend` over TCP Port 3306)

## Troubleshooting Process (How I Figured it Out)

### 1. Verification of Labels and NetworkPolicy 
First, I verified the current Kubernetes labels attached to the `backend` and `mysql` pods using `kubectl get pods -n netpol --show-labels`. 
- `backend` correctly had `role=backend`.
- `mysql` correctly had `name=mysql`.
- The `db-test` NetworkPolicy `podSelector` and ingress `matchLabels` were perfectly aligned to match the pods. 

### 2. Testing direct communication to rule out the firewall
I executed an interactive shell into the `backend` pod and tested a connection directly to the **Service IP** (`db`):
- `telnet db 3306` &rarr; `Connection timed out`.

Then, I identified the direct native **Pod IP** of `mysql` (`10.40.0.4`) and tested the connection skipping the cluster service (`kube-proxy`) layer:
- `telnet 10.40.0.4 3306` &rarr; `Connected successfully`.

**Conclusion**: The NetworkPolicy provided by Weave Net *was working and was actively allowing the connection* natively! The issue strictly lived with how the traffic was being handled when directed to the `db` Service IP.

### 3. Identifying SNAT (Source NAT) Configuration
Since direct Pod-to-Pod traffic worked, but Pod-to-Service traffic failed, it strongly implied that `kube-proxy` was rewriting the IP packet source when traversing via the service interface. If `kube-proxy` performed Source NAT (Masquerading), the packet arriving at the `mysql` Pod would appear to come from the Host Node, NOT the `backend` pod. Consequently, Weave Net would drop the packets, because the Host Node lacks the `role: backend` label.

To confirm this, I inspected the `kube-proxy` configuration generated during the cluster spin up. 
Command: `kubectl get cm kube-proxy -n kube-system -o yaml`

### 4. Root Cause Discovered
Inside the configuration, `clusterCIDR` was set to `10.244.0.0/16`. This is a classic default (usually provided for Flannel or generic `kubeadm` installations). 

However, we are using **Weave Net**, which assigns IPs natively out of the `10.32.0.0/12` block. The `backend` pod's real IP was `10.46.0.1` and `mysql` was `10.40.0.4`.

Because `10.46.0.1` falls **outside** of `10.244.0.0/16`, `kube-proxy` automatically assumed the traffic originated outside the internal cluster network and "fixed" it by overwriting the source IP of the packet with the Worker Node's IP. 

Once the Source IP changed, the `db-test` NetworkPolicy immediately rejected the traffic.

## The Fix

1. I edited the `kube-proxy` ConfigMap in the `kube-system` namespace.
2. I modified `clusterCIDR: 10.244.0.0/16` to strictly match Weave Net's defaults: `clusterCIDR: 10.32.0.0/12`.
3. I safely restarted the kube-proxy layer by terminating all associated pods, forcing them to respawn using the updated ConfigMap (`kubectl delete pods -n kube-system -l k8s-app=kube-proxy`).

Following the restart, `kube-proxy` stopped masquerading internal service traffic, preserved original Pod IPs, and `backend` successfully resumed communications with `db`, while correctly continuing to drop traffic stemming from the `frontend`.
