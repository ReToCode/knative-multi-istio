# Testing clashing gateway configs

## Test case
```bash
# Install based on README.md in root folder until 1-single-istio

# Test before adding clashing gateway
curl -k https://svc-always-scaled-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/
Hello Serverless!

oc apply -f .

# Still works
curl -k https://svc-always-scaled-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/
Hello Serverless!

# But basic k8s workload does not work
curl -kiv https://hello-k8s.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com
* Connection state changed (MAX_CONCURRENT_STREAMS == 2147483647)!
< HTTP/2 404 
HTTP/2 404 
< date: Fri, 10 Mar 2023 09:38:40 GMT
date: Fri, 10 Mar 2023 09:38:40 GMT
< server: istio-envoy
server: istio-envoy

# Delete the knative-ingress-gateway
oc delete -f ../1-single-istio/gateways.yaml

# basic k8s workload then works
curl -k https://hello-k8s.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com
Hello native k8s workload

# The same applies the other way around
```

## Istioctl output
```bash
# When basic k8s workload works:
istioctl proxy-config listeners -n istio-system  istio-ingressgateway-bc446949b-szws6
ADDRESS PORT  MATCH DESTINATION
0.0.0.0 8081  ALL   Route: http.8081
0.0.0.0 8443  ALL   Route: https.443.https.clashing-ingress-gateway.clash
0.0.0.0 15021 ALL   Inline Route: /healthz/ready*
0.0.0.0 15090 ALL   Inline Route: /stats/prometheus*

# When knative works
 istioctl proxy-config listeners -n istio-system  istio-ingressgateway-bc446949b-szws6
ADDRESS PORT  MATCH DESTINATION
0.0.0.0 8081  ALL   Route: http.8081
0.0.0.0 8443  ALL   Route: https.443.https.knative-ingress-gateway.knative-serving
0.0.0.0 15021 ALL   Inline Route: /healthz/ready*
0.0.0.0 15090 ALL   Inline Route: /stats/prometheus*
```
