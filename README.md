# agentgateway as an Istio Ambient Egress Gateway

![Overview](overview.png)

This repository demonstrates how to run Gloo Gateway with agentgateway proxy as an egress gateway for Istio traffic, including external authentication with source identity and Dynamic Forward Proxy (DFP) with CONNECT to an external proxy.

---

## Install Gloo Mesh Istio

```bash
export GLOO_MESH_LICENSE_KEY=<license_key>

export ISTIO_VERSION=1.28.1
export ISTIO_IMAGE=${ISTIO_VERSION}-solo
export REPO=us-docker.pkg.dev/soloio-img/istio
export HELM_REPO=oci://us-docker.pkg.dev/soloio-img/istio-helm
```

CRDs:
```
helm upgrade --install istio-base ${HELM_REPO}/base \
--namespace istio-system \
--create-namespace \
--version ${ISTIO_IMAGE} \
-f - <<EOF
defaultRevision: ""
profile: ambient
EOF
```

Istiod:
```
helm upgrade --install istiod ${HELM_REPO}/istiod \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
env:
  # Assigns IP addresses to multicluster services
  PILOT_ENABLE_IP_AUTOALLOCATE: "true"
  # Disable selecting workload entries for local service routing.
  # Required for Gloo VirtualDestinaton functionality.
  # PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
  # Required when meshConfig.trustDomain is set
  PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
global:
  hub: ${REPO}
  network: cluster1
  proxy:
    clusterDomain: cluster.local
  tag: ${ISTIO_IMAGE}
istio_cni:
  namespace: istio-system
  enabled: true
meshConfig:
  accessLogFile: /dev/stdout
  defaultConfig:
    proxyMetadata:
      ISTIO_META_DNS_AUTO_ALLOCATE: "true"
      ISTIO_META_DNS_CAPTURE: "true"
# Required to enable multicluster support
platforms:
  peering:
    enabled: true
profile: ambient
license:
    value: ${GLOO_MESH_LICENSE_KEY}
EOF
```

CNI:
```
helm upgrade --install istio-cni ${HELM_REPO}/cni \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
# Assigns IP addresses to multicluster services
ambient:
  dnsCapture: true
excludeNamespaces:
  - istio-system
  - kube-system
global:
  hub: ${REPO}
  tag: ${ISTIO_IMAGE}
  variant: distroless
  platform: gke # UNCOMMENT FOR GKE
profile: ambient
EOF
```

ztunnel:
```
helm upgrade --install ztunnel ${HELM_REPO}/ztunnel \
--namespace istio-system \
--version ${ISTIO_IMAGE} \
-f - <<EOF
configValidation: true
enabled: true
env:
  L7_ENABLED: "true"
  # Required when a unique trust domain is set for each cluster
  SKIP_VALIDATE_TRUST_DOMAIN: "true"
hub: ${REPO}
tag: ${ISTIO_IMAGE}
istioNamespace: istio-system
namespace: istio-system
network: cluster1
profile: ambient
proxy:
  clusterDomain: cluster.local
terminationGracePeriodSeconds: 29
variant: distroless
egressPolicies:
- namespaces:
  - common-infrastructure
  policy: Passthrough
- gateway: egress-gateway.common-infrastructure.svc.cluster.local
  matchCidrs:
  - 0.0.0.0/0
  - ::/0
  policy: Gateway
EOF
```

---

## Install Gloo Gateway with agentgateway

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/experimental-install.yaml

helm upgrade -i gloo-gateway-crds oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway-crds \
--create-namespace \
--namespace gloo-system \
--version 2.1.0-beta.1

helm upgrade -i gloo-gateway oci://us-docker.pkg.dev/solo-public/gloo-gateway/charts/gloo-gateway \
-n gloo-system \
--version 2.1.0-beta.1 \
--set agentgateway.enabled=true \
--set-string licensing.glooGatewayLicenseKey=$GLOO_GATEWAY_LICENSE_KEY \
--set-string licensing.agentgatewayLicenseKey=$AGENTGATEWAY_LICENSE_KEY
```


## Deploy Squid Proxy

This proxy can be external to the cluster but for the sake of this demo, we're going to deploy it in the cluster as a Deployment/Service. See squid-deployment.yml for details.

```bash
kubectl apply -f squid-deployment.yml
```
---

## agentgateway Egress Gateway

### Create Namespace for the Egress Gateway

```bash
kubectl create ns common-infrastructure
kubectl label ns common-infrastructure istio.io/use-waypoint=egress-gateway
```

### Deploy the Egress Gateway with Dynamic Forward Proxy

```bash
kubectl apply -f agentgateway.yaml
```

We're ready to test. Istio will send all unmatched traffic to the egress gateway (See the ztunnel helm chart for configuration). The egress gateway will then do any ext-auth checks and then forward to the squid proxy using CONNECT. 

## Deploy Sample Application in ns1 Namespace

```bash
kubectl create ns ns1
kubectl label ns ns1 istio.io/dataplane-mode=ambient
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/sleep/sleep.yaml -n ns1
```

Test connectivity:

```bash
kubectl exec deploy/sleep -n ns1 -- curl httpbingo.org/headers
```
You should see a json output with header information.

Check logs in the Egress Gateway:

```bash
kubectl logs deploy/egress-gateway -n common-infrastructure
```

Example output:

```
2025-12-05T20:10:43.445524Z     info    request gateway=common-infrastructure/egress-gateway listener=waypoint/common-infrastructure/egress-gateway route=waypoint-default-dfp endpoint=httpbingo.org:80 src.addr=10.40.3.52:39116 http.method=GET http.host=httpbingo.org http.path=/headers http.version=HTTP/1.1 http.status=200 protocol=http duration=553ms
```

Check logs in the Squid Proxy:

```bash
kubectl logs deploy/squid-proxy -n common-infrastructure
```

Example output:
```
1764965448.445   5552 10.40.3.47 TCP_TUNNEL/200 811 CONNECT httpbingo.org:80 - HIER_DIRECT/66.241.125.232 -
```

---


## External Auth Server

```bash
kubectl apply -f ext-auth-server.yml
```

This ext auth server only allows connects with the header 
Test connectivity:

```bash
kubectl exec deploy/sleep -n ns1 -- curl httpbingo.org/headers
```

You should see `external authorization failed`


Try again with the allowed header:


You should see an extra header:

```bash
kubectl exec deploy/sleep -n ns1 -- curl httpbingo.org/headers -H "x-ext-authz: allow"
```

You should see a successful result, with an extra header:
```
"X-Ext-Authz-Check-Result": [
  "allowed"
],
```

Check the Ext Auth Server logs:

```bash
kubectl logs deploy/ext-authz -n common-infrastructure
```

Example output:

```
2025/12/05 20:13:53 [gRPCv3][allowed]: httpbingo.org/headers, attributes: source:{address:{socket_address:{address:"10.40.3.52"  port_value:39116}}  principal:"spiffe://cluster.local/ns/ns1/sa/sleep"}  destination:{address:{socket_address:{address:"10.40.3.47"  port_value:15008}}}  request:{time:{seconds:1764965285  nanos:166653674}  http:{id:"4086db120dc5b2cf77c103853d8a28b8"  method:"GET"  headers:{key:"accept"  value:"*/*"}  headers:{key:"user-agent"  value:"curl/8.16.0"}  headers:{key:"x-ext-authz"  value:"allow"}  path:"/headers"  host:"httpbingo.org"  scheme:"https"  protocol:"HTTP/1.1"}}  tls_session:{}
```

Note that you have access to the client identity (`spiffe://cluster.local/ns/ns1/sa/sleep`), which allows you to make auth decisions dynamically.

---

Happy egressing! 🚀