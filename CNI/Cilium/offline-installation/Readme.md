

# 1- Cilium installaton 

## 1- Install Clium CRD First 
```
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumbgpadvertisements.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumbgpclusterconfigs.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumbgpnodeconfigs.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumbgppeerconfigs.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumcidrgroups.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumclusterwidenetworkpolicies.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumegressgatewaypolicies.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumendpoints.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumenvoyconfigs.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumidentities.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumloadbalancerippools.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumlocalredirectpolicies.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliuml2announcementpolicies.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumnetworkpolicies.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumnodeconfigs.yaml

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.19.3/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumnodes.yaml
```

# 2- Customize Cilium values
# Values.yml 
```
global:
  registey: kube.aradarpanet.ir

# L2 advertise & KPR
kubeProxyReplacement: true

l2announcements:
  enabled: true

externalIPs:
  enabled: true

# --- Core networking ---

k8sServiceHost: 192.168.100.16
k8sServicePort: 6443

# --- Cilium Agent ---
image:
  repository: kube.aradarpanet.ir/cilium/cilium
  tag: v1.19.3
  pullPolicy: IfNotPresent
  useDigest: false

# --- Operator (generic chart) ---

operator:
  replicas: 2

  image:
    repository: kube.aradarpanet.ir/cilium/operator
    tag: v1.19.3
    pullPolicy: IfNotPresent
    useDigest: false



# --- Envoy ( L7 / ingress / gateway) ---
envoy:
  enabled: true
  image:
    repository: kube.aradarpanet.ir/cilium/cilium-envoy
    tag: v1.34.10-1762597008-ff7ae7d623be00078865cff1b0672cc5d9bfc6d5
    pullPolicy: IfNotPresent
    useDigest: false

# --- Certgen ---
certgen:
  image:
    repository: kube.aradarpanet.ir/cilium/certgen
    tag: v0.2.4
    pullPolicy: IfNotPresent
    useDigest: false

# --- Hubble (observability) ---
hubble:
  enabled: true

  relay:
    enabled: true
    image:
      repository: kube.aradarpanet.ir/cilium/hubble-relay
      tag: v1.19.3
      pullPolicy: IfNotPresent
      useDigest: false

  ui:
    enabled: true

    frontend:
      image:
        repository: kube.aradarpanet.ir/cilium/hubble-ui
        tag: v0.13.3
        pullPolicy: IfNotPresent
        useDigest: false

    backend:
      image:
        repository: kube.aradarpanet.ir/cilium/hubble-ui-backend
        tag: v0.13.3
        pullPolicy: IfNotPresent
        useDigest: false

# --- CNI install ---
cni:
  binPath: /opt/cni/bin
  confPath: /etc/cni/net.d


#  network:
enableIPv4Masquerade: true


# --- BPF / performance ---
bpf:
  masquerade: true

# --- kube-proxy replacement health ---
kubeProxyReplacementHealthzBindAddr: "0.0.0.0:10256"

# --- Security ---
securityContext:
  privileged: true

# --- Node init  ---
nodeinit:
  enabled: false

# --- clustermesh ---
clustermesh:
  useAPIServer: false

# --- TLS ---
tls:
  enabled: false

# --- Auth ---
authentication:
  mutual:
    spire:
      enabled: false

# --- image pull ---
imagePullSecrets: []
```

# 3- install cilium 
```
helm repo add arad https://registry.aradarpanet.ir/repository/Helm-Hosted/
helm repo update 
 helm upgrade  cilium arad/cilium   -n kube-system   --create-namespace   -f values.yml
```
