# Install Istio

1. Add Istio helm chart repository:

```
export ISTIO_VERSION="1.3.6"
helm repo add istio.io https://storage.googleapis.com/istio-release/releases/${ISTIO_VERSION}/charts/
helm repo update
```
2. Install CRDs for Istio:

```
kubectl create namespace istio-system
helm install istio-init istio.io/istio-init --wait --namespace istio-system --version ${ISTIO_VERSION}
kubectl -n istio-system wait --for=condition=complete job --all
```
3. Label Istio namespace and which will trigger kubed to copy there the secret with certificates signed by Let's Encrypt:

```
kubectl label namespace istio-system app=kubed
```

4. Install Istio:

(make sure to run the command below on bash)

```
helm install istio istio.io/istio --wait --namespace istio-system --version ${ISTIO_VERSION} \
  --set gateways.istio-ingressgateway.autoscaleMax=1 \
  --set gateways.istio-ingressgateway.autoscaleMin=1 \
  --set gateways.istio-ingressgateway.ports[0].name=status-port \
  --set gateways.istio-ingressgateway.ports[0].port=15020 \
  --set gateways.istio-ingressgateway.ports[0].targetPort=15020 \
  --set gateways.istio-ingressgateway.ports[1].name=http \
  --set gateways.istio-ingressgateway.ports[1].nodePort=31380 \
  --set gateways.istio-ingressgateway.ports[1].port=80 \
  --set gateways.istio-ingressgateway.ports[1].targetPort=80 \
  --set gateways.istio-ingressgateway.ports[2].name=https \
  --set gateways.istio-ingressgateway.ports[2].nodePort=31390 \
  --set gateways.istio-ingressgateway.ports[2].port=443 \
  --set gateways.istio-ingressgateway.ports[3].name=ssh \
  --set gateways.istio-ingressgateway.ports[3].nodePort=31400 \
  --set gateways.istio-ingressgateway.ports[3].port=22 \
  --set gateways.istio-ingressgateway.sds.enabled=true \
  --set global.disablePolicyChecks=true \
  --set global.k8sIngress.enableHttps=true \
  --set global.k8sIngress.enabled=true \
  --set global.proxy.autoInject=disabled \
  --set grafana.datasources."datasources\.yaml".datasources[0].access=proxy \
  --set grafana.datasources."datasources\.yaml".datasources[0].editable=true \
  --set grafana.datasources."datasources\.yaml".datasources[0].isDefault=true \
  --set grafana.datasources."datasources\.yaml".datasources[0].jsonData.timeInterval=5s \
  --set grafana.datasources."datasources\.yaml".datasources[0].name=Prometheus \
  --set grafana.datasources."datasources\.yaml".datasources[0].orgId=1 \
  --set grafana.datasources."datasources\.yaml".datasources[0].type=prometheus \
  --set grafana.datasources."datasources\.yaml".datasources[0].url=http://prometheus-system-np.knative-monitoring.svc.cluster.local:8080 \
  --set grafana.enabled=true \
  --set kiali.contextPath=/ \
  --set kiali.createDemoSecret=true \
  --set kiali.dashboard.grafanaURL=http://grafana.${MY_DOMAIN}/ \
  --set kiali.dashboard.jaegerURL=http://jaeger.${MY_DOMAIN}/ \
  --set kiali.enabled=true \
  --set kiali.prometheusAddr=http://prometheus-system-np.knative-monitoring.svc.cluster.local:8080 \
  --set mixer.adapters.prometheus.enabled=false \
  --set pilot.traceSampling=100 \
  --set prometheus.enabled=false \
  --set sidecarInjectorWebhook.enableNamespacesByDefault=true \
  --set sidecarInjectorWebhook.enabled=true \
  --set tracing.enabled=true
```

Output:

```
NAME: istio
LAST DEPLOYED: Sun Apr 12 12:36:44 2020
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing Istio.

Your release is named Istio.

To get started running application with Istio, execute the following steps:
1. Label namespace that application object will be deployed to by the following command (take default namespace as an example)

$ kubectl label namespace default istio-injection=enabled
$ kubectl get namespace -L istio-injection

2. Deploy your applications

$ kubectl apply -f <your-application>.yaml

For more information on running Istio, visit:
https://istio.io/
```

5. Let istio-ingressgateway to use cert-manager generated certificate via SDS. Steps are taken from this URL: https://istio.io/docs/tasks/traffic-management/ingress/ingress-certmgr/.

```
kubectl -n istio-system patch gateway istio-autogenerated-k8s-ingress \
  --type=json \
  -p="[{"op": "replace", "path": "/spec/servers/1/tls", "value": {"credentialName": "ingress-cert-${LETSENCRYPT_ENVIRONMENT}", "mode": "SIMPLE", "privateKey": "sds", "serverCertificate": "sds"}}]"
```

6. Disable HTTP2 for gateway istio-autogenerated-k8s-ingress to be compatible with Knative:

```
kubectl -n istio-system patch gateway istio-autogenerated-k8s-ingress --type=json \
  -p="[{"op": "replace", "path": "/spec/servers/0/port", "value": {"name": "http", "number": "80", "protocol": "HTTP"}}]"

```

7. Allow the default namespace to use Istio injection:

```
kubectl label namespace default istio-injection=enabled
```

8. Configure the Istio services Jaeger and Kiali to be visible externally:

```
cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-services-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http-istio-services
      protocol: HTTP
    hosts:
    - grafana-istio.${MY_DOMAIN}
    - jaeger-istio.${MY_DOMAIN}
    - kiali-istio.${MY_DOMAIN}
  - port:
      number: 443
      name: https-istio-services
      protocol: HTTPS
    hosts:
    - grafana-istio.${MY_DOMAIN}
    - jaeger-istio.${MY_DOMAIN}
    - kiali-istio.${MY_DOMAIN}
    tls:
      credentialName: ingress-cert-${LETSENCRYPT_ENVIRONMENT}
      mode: SIMPLE
      privateKey: sds
      serverCertificate: sds
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana-istio-virtual-service
  namespace: istio-system
spec:
  hosts:
  - grafana-istio.${MY_DOMAIN}
  gateways:
  - istio-services-gateway
  http:
  - route:
    - destination:
        host: grafana.istio-system.svc.cluster.local
        port:
          number: 3000
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: jaeger-istio-virtual-service
  namespace: istio-system
spec:
  hosts:
  - jaeger-istio.${MY_DOMAIN}
  gateways:
  - istio-services-gateway
  http:
  - route:
    - destination:
        host: tracing.istio-system.svc.cluster.local
        port:
          number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kiali-istio-virtual-service
  namespace: istio-system
spec:
  hosts:
  - kiali-istio.${MY_DOMAIN}
  gateways:
  - istio-services-gateway
  http:
  - route:
    - destination:
        host: kiali.istio-system.svc.cluster.local
        port:
          number: 20001
EOF
```

## Create DNS records

1. Install external-dns and let it manage mylabs.dev entries in Route 53 (Do not upgrade external-dns, because it's not backward compatible and using different way of authentication to Route53 using roles):

```
kubectl create namespace external-dns
helm install external-dns stable/external-dns --namespace external-dns --version 2.10.1 --wait \
  --set aws.credentials.accessKey="${USER_AWS_ACCESS_KEY_ID}" \
  --set aws.credentials.secretKey="${USER_AWS_SECRET_ACCESS_KEY}" \
  --set aws.region=us-east-1 \
  --set domainFilters={${MY_DOMAIN}} \
  --set istioIngressGateways={istio-system/istio-ingressgateway} \
  --set interval="10s" \
  --set policy="sync" \
  --set rbac.create=true \
  --set sources="{istio-gateway,service}" \
  --set txtOwnerId="${USER}-k8s.${MY_DOMAIN}"
```

output 

```
namespace/external-dns created
NAME: external-dns
LAST DEPLOYED: Fri Dec 27 10:53:29 2019
NAMESPACE: external-dns
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

To verify that external-dns has started, run:

  kubectl --namespace=external-dns get pods -l "app.kubernetes.io/name=external-dns,app.kubernetes.io/instance=external-dns"
```