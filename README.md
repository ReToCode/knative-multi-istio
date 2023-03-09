# Knative on a multi-istio cluster
This repository contains findings around installing `Knative` on a cluster where multiple `istio control planes` are present.

## Prerequisites
* An OpenShift cluster

## Conditions / boundaries
* `Serverless` can only target ONE istio service mesh. Multiple meshes can be present in the cluster, but `Serverless` will only be available on one of them.
* The mesh that `Serverless` is part of has to be distinct and only for `Serverless` workloads, as additional configuration like gateways might interfere with the automated mesh configuration by `Serverless`.
* Cluster external serverless-services are expected to be called via `OpenShift ingress` using `OpenShift Routes`.

## Reference setup
```bash
# Install Operators
oc apply -f 0-prerequisites

# Install single-mesh setup
oc apply -f 1-single-istio

# Install a ksvc and test it
oc apply -f service

curl -k https://svc-always-scaled-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/
Hello Serverless!
```

## Multi-istio setup
```bash
# Install Operators
oc apply -f 0-prerequisites

# Install single-mesh setup
oc apply -f 1-single-istio

# Remove the knative-local-gateway in istio-system
oc delete -f 1-single-istio/knative-local-gateway-service.yaml

# Remove the ksvc, to be recreated in the new mesh
oc delete -f service

# Install the second mesh
oc apply -f 2-multi-istio/smmr-istio-system.yaml
oc apply -f 2-multi-istio

# Install a ksvc and test it
oc apply -f service

curl -k https://svc-always-scaled-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/
Hello Serverless!
```

## Relevant changes

**Gateway selector**
```yaml
# single-istio
selector:
  istio: ingressgateway
# multi-istio
selector:
  knative: ingressgateway
```

**Knative local gateway service selector (optional)**
```yaml
# single-istio
selector:
  istio: ingressgateway
# multi-istio
selector:
  knative: ingressgateway
```

**Override knative serving istio location**
```yaml
spec:
  config:
    istio:
      gateway.knative-serving.knative-ingress-gateway: istio-ingressgateway.istio-system-reto.svc.cluster.local
      local-gateway.knative-serving.knative-local-gateway: knative-local-gateway.istio-system-reto.svc.cluster.local
```

**SMCP add custom label to ingress-gateway**
```yaml
apiVersion: maistra.io/v2
kind: ServiceMeshControlPlane
metadata:
  name: basic
  namespace: istio-system-reto
spec:
  gateways:
    ingress:
      service:
        metadata:
          labels:
            knative: ingressgateway
```

**Results in a diff on KIngress**
```yaml
# single-istio
  privateLoadBalancer:
    ingress:
    - domainInternal: knative-local-gateway.istio-system.svc.cluster.local
  publicLoadBalancer:
    ingress:
    - domainInternal: istio-ingressgateway.istio-system.svc.cluster.local    
# multi-istio
  privateLoadBalancer:
    ingress:
    - domainInternal: knative-local-gateway.istio-system-reto.svc.cluster.local
  publicLoadBalancer:
    ingress:
    - domainInternal: istio-ingressgateway.istio-system-reto.svc.cluster.local
```

