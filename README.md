# Istio Top-level Mesh Design

![Mesh-v3.drawio.svg](assets/Mesh-v3.drawio.svg)

# Prerequisites

- `istioctl` - https://istio.io/latest/docs/setup/install/istioctl/

# Installation

## Versions

- `istioctl` - `v1.27.1`
- `istio` - `v1.27.1`

## Control Plane

- Create a _Kubernetes_ cluster dedicated for **Istio**
    - A _VPC_ dedicated for **Istio** cluster
    - Atleast a single worker `node` to accommodate `istiod` and `gateways`(_ingress, egress, eastwest_)
- Create a `namespace` dedicated for `istio` (*istio-system*).
    - With corresponding `topology.istio.io/network`label matching the `network` specified in the  `istiod.yaml` .
- Deploy `istiod` and its `eastwest` component in a *MultiCluster Primary-Remote in Different Network* setup using `istioctl`.

```yaml
# istiod.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
  meshConfig:
    accessLogFile: /dev/stdout
  values:
    global:
      meshID: mesh-1
      multiCluster:
        clusterName: istio-mesh
      network: <istiod-network>
      externalIstiod: true
```

```yaml
# eastwest.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: eastwest
spec:
  revision: ""
  profile: empty
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: <istiod-network>
        enabled: true
        k8s:
          env:
            # traffic through this gateway should be routed inside the network
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: <istiod-network>
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
  values:
    gateways:
      istio-ingressgateway:
        injectionTemplate: gateway
    global:
      network: <istiod-network>
```

- Expose `istiod` and its `eastwest` gateway via `virtual-service`.

```yaml
# istiod-vs.yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: istiod-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        name: tls-istiod
        number: 15012
        protocol: tls
      tls:
        mode: PASSTHROUGH        
      hosts:
        - "*"
    - port:
        name: tls-istiodwebhook
        number: 15017
        protocol: tls
      tls:
        mode: PASSTHROUGH          
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: istiod-vs
spec:
  hosts:
  - "*"
  gateways:
  - istiod-gateway
  tls:
  - match:
    - port: 15012
      sniHosts:
      - "*"
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local
        port:
          number: 15012
  - match:
    - port: 15017
      sniHosts:
      - "*"
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local
        port:
          number: 443
```

## Data Plane

- Create a `namespace` dedicated for `istio` (*istio-system*).
    - With corresponding `topology.istio.io/network`label matching the `network` specified in the  `istiod.yaml` .
    - With corresponding **`topology.istio.io/controlPlaneClusters` annotation matching the `meshID` of the *Service Mesh* specified in `istiod.yaml` of the *Control Plane*.**
- Deploy `istiod` and its `eastwest` component in a *MultiCluster Primary-Remote in Different Network* setup using `istioctl`.

```yaml
# istiod.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: remote
  values:
    istiodRemote:
      injectionPath: /inject/cluster/<remote-cluster>/net/<remote-network>
    global:
      # EastWest Gateway IP
      remotePilotAddress: <istiod-eastwest-ip>
```

```yaml
# eastwest.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: eastwest
spec:
  revision: ""
  profile: empty
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: <remote-network>
        enabled: true
        k8s:
          env:
            # traffic through this gateway should be routed inside the network
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: <remote-network>
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
  values:
    gateways:
      istio-ingressgateway:
        injectionTemplate: gateway
    global:
      network: <remote-network>
```

- Create a `remote-secret` of the *Data Plane* cluster and apply it to the *Control Plane* cluster using `istioctl`.
- Expose its `eastwest` gateway via `gateway` .

```yaml
# cross-network-gateway.yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: cross-network-gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.local"
```

## Virtual Machine

- Create a `namespace` on all members of the mesh and a `service-account` in the *Control Plane* for the *VM* to be connected to the mesh.
- Apply `workload-group` manifest matching the details of the *VM*.

```yaml
# workload-group.yaml
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
  name: <workload-group-name>
  namespace: <workload-group-ns>
spec:
  metadata:
    labels:
      app.kubernetes.io/cluster: <istiod-cluster>
      app.kubernetes.io/component: <component-name>
  template:
    serviceAccount: <workload-sa-name>
    network: <workload-network>
```

- Configure the `workload-entry` using `istioctl` accompanied by the `workload-group` manifest.
- Transfer all the generated credentials to the *VM* and follow the guide [referred](https://www.notion.so/Istio-Setup-284b56b8bc61806fb270c3d148bea986?pvs=21).
- Create a `service` matching the *VM*’s specifications to access the `VM` from the cluster.

```yaml
# t2-replica.yaml
apiVersion: v1
kind: Service
metadata:
  name: <workload-svc-name>
  labels:
    app.kubernetes.io/cluster: <istiod-cluster>
    app.kubernetes.io/component: <component-name>
spec:
  selector:
    app.kubernetes.io/cluster: <istiod-cluster>
    app.kubernetes.io/component: <component-name>
  ports:
    - protocol: TCP
      name: tcp-<svc>
      port: <port-number>
      targetPort: <port-number>
```

# Setup

- All members of the mesh should have a:
    - `namespace` *label* of `istio-injection: enabled`, or
    - `pod` *label* of `sidecar.istio.io/inject: "true"`.
- Strictly follows *Namespace Sameness* concept.
- Install `ufw` firewall module.

```bash
# UFW Guide:
## Defaults
ufw default allow outgoing
ufw default deny incoming

## Rules
ufw allow <in[coming]|out[going]> from <ip|cidr|"any"> to <ip|cidr|"any"> port <num> proto <tcp|udp>

## Mode
ufw <enable|disable|reload>

## Status
ufw status <verbose|numbered>
```

- Restrict `ssh` access to `workload-entry` only from the default VPC; i.e `10.130.0.0/16`.
- *Inbound* port of *IstioEnvoy* (`15006`) should be open to and allowed incoming from subscribed *Kubernetes* `node`s.

# References

[Deployment Models](https://istio.io/latest/docs/ops/deployment/deployment-models/)

[Install Primary-Remote on different networks](https://istio.io/latest/docs/setup/install/multicluster/primary-remote_multi-network/)

[Virtual Machine Installation](https://istio.io/latest/docs/setup/install/virtual-machine/)

[Installing the Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)

[Application Requirements](https://istio.io/latest/docs/ops/deployment/application-requirements/)
