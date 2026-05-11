# Cilium L2 Announcement on Kubernetes

A production-style guide for exposing Kubernetes `LoadBalancer` services on Layer 2 networks using Cilium L2 Announcements.

This repository demonstrates how to configure Cilium L2 Announcements in a Kubernetes cluster to advertise service IPs directly on the local network without requiring external load balancers such as MetalLB.

---

# Overview

Cilium L2 Announcements enable Kubernetes services to become reachable from the local network by responding to ARP/NDP requests for `LoadBalancer` or `ExternalIP` services.

This approach is commonly used in:

- Bare-metal Kubernetes clusters
- Homelab environments
- On-premise datacenters
- Edge Kubernetes deployments
- Lab and development clusters

Unlike NodePort-based exposure, L2 Announcements allow services to use dedicated virtual IPs while maintaining transparent failover between nodes.

Official Documentation:  
https://docs.cilium.io/en/stable/network/l2-announcements/

---

# Architecture

```text
                    +----------------------+
                    |   External Clients   |
                    +----------+-----------+
                               |
                         ARP/NDP Requests
                               |
                +--------------+--------------+
                | Kubernetes Cluster Network  |
                +--------------+--------------+
                               |
                   +-----------+-----------+
                   |  Leader Node Selected |
                   |   by Cilium Lease     |
                   +-----------+-----------+
                               |
                     LoadBalancer Service
                               |
                     Kubernetes Workloads
```

---

# Repository Structure

```bash
.
├── cilium-l2-policy.yaml
├── ip-pool.yaml
├── nginx-demo.yaml
├── values.yaml
└── README.md
```

| File | Description |
|---|---|
| `values.yaml` | Helm values for enabling Cilium L2 Announcements |
| `ip-pool.yaml` | Defines the IP pool used for LoadBalancer services |
| `cilium-l2-policy.yaml` | Announces service IPs on selected interfaces |
| `nginx-demo.yaml` | Example workload exposed through LoadBalancer |
| `README.md` | Documentation and deployment guide |

---

# Prerequisites

Before using this setup, ensure the following requirements are met:

- Kubernetes cluster installed
- Cilium installed as the CNI
- kube-proxy replacement enabled
- Nodes connected to the same Layer 2 network
- Available subnet/IP range for LoadBalancer services
- Helm installed

Supported environments include:

- kubeadm
- Talos Linux
- K3s
- RKE2
- Kind (with networking adjustments)
- Bare-metal clusters

---

# Key Concepts

## L2 Announcements

Cilium answers ARP/NDP requests on behalf of Kubernetes services.

Only one node announces a service IP at a time using Kubernetes lease-based leader election.

---

## Lease-Based Failover

Each advertised service creates a Kubernetes Lease object.

If the active node fails, another node takes ownership and begins announcing the IP.

---

## LoadBalancer IP Pool

Cilium allocates IPs from a configured pool to Kubernetes `LoadBalancer` services.

---

# Enable Cilium L2 Announcements

## Helm Configuration

Example Helm values:

```yaml
kubeProxyReplacement: true

l2announcements:
  enabled: true

externalIPs:
  enabled: true

k8sClientRateLimit:
  qps: 50
  burst: 100
```

Install or upgrade Cilium:

```bash
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  -f values.yaml
```

---

# Configure LoadBalancer IP Pool

Create a `CiliumLoadBalancerIPPool` resource.

## Example

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
spec:
  blocks:
    - start: 192.168.1.240
      stop: 192.168.1.250
```

Apply configuration:

```bash
kubectl apply -f ip-pool.yaml
```

---

# Configure L2 Announcement Policy

The policy determines:

- Which services are announced
- Which nodes can announce them
- Which network interfaces are used

## Example

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: l2-policy
spec:
  serviceSelector:
    matchLabels:
      expose: "true"

  interfaces:
    - ^eth0$

  externalIPs: true
  loadBalancerIPs: true
```

Apply configuration:

```bash
kubectl apply -f cilium-l2-policy.yaml
```

---

# Deploy Example Workload

Example NGINX deployment:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    expose: "true"
spec:
  type: LoadBalancer

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80
```

Deploy workload:

```bash
kubectl apply -f nginx-demo.yaml
```

---

# Validate Configuration

## Verify LoadBalancer IP

```bash
kubectl get svc
```

Expected output:

```bash
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP
nginx   LoadBalancer   10.96.120.10    192.168.1.240
```

---

## Check L2 Leases

```bash
kubectl -n kube-system get lease
```

Example:

```bash
cilium-l2announce-default-nginx
```

The holder node is currently announcing the service IP.

---

## Verify ARP Advertisement

From another machine on the same network:

```bash
arp -an | grep 192.168.1.240
```

Or test connectivity:

```bash
curl http://192.168.1.240
```

---

# Failover Behavior

If the active node becomes unavailable:

1. Lease expires
2. Another node acquires leadership
3. ARP replies begin from the new node
4. Traffic resumes automatically

Default failover timing is typically between 10–20 seconds depending on lease settings.

---

# Troubleshooting

## Service Has No External IP

Verify:

```bash
kubectl get ippools
```

Ensure the pool contains available IP addresses.

---

## L2 Lease Not Created

Inspect logs:

```bash
kubectl -n kube-system logs ds/cilium | grep l2
```

Common causes:

- Missing RBAC permissions
- Incorrect policy selectors
- kube-proxy replacement disabled

---

## ARP Not Responding

Check:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg shell -- db/show devices
```

Ensure the selected interface matches your network device.

---

## Connectivity Works Intermittently

Potential causes:

- Lease renewal timing
- Low API rate limits
- Incorrect interface regex
- Network segmentation

Recommended tuning:

```yaml
k8sClientRateLimit:
  qps: 100
  burst: 200
```

---

# Best Practices

## Use Dedicated IP Ranges

Allocate a subnet exclusively for Kubernetes LoadBalancer services.

---

## Avoid `externalTrafficPolicy: Local`

Cilium L2 Announcements may drop traffic if services are announced from nodes without backend pods.

---

## Monitor Lease Stability

Monitor Kubernetes leases and Cilium agent logs for leader flapping or renewal failures.

---

## Prefer BGP for Large Environments

L2 Announcements are ideal for smaller Layer 2 domains.

Large-scale environments should consider:

- Cilium BGP Control Plane
- External routers
- EVPN/VXLAN architectures

---

# Security Considerations

- Restrict announcement policies to required interfaces only
- Use node selectors to limit announcement nodes
- Avoid exposing internal services unintentionally
- Monitor ARP behavior in shared VLAN environments

---

# References

- https://docs.cilium.io/en/stable/network/l2-announcements/
- https://github.com/cilium/cilium
- https://kubernetes.io/docs/home/

---






