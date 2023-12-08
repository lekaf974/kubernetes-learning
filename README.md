# Test of Kubernetes Gateway API

## Who am I

<!-- Presenting myself, I am Matthieu a 40's years old father of 2 beautiful girls and married with the best wife (Yes I am lucky!).
I started to work in computer science for a while now (about 20 years) where I had several roles like analyst programer, database administrator, linux system administrator that make me moving to devops/platform engineering. This make me starting the journey using Kubernetes and all the ecosystem around it.   -->

After using default ingress for years, I decided to test and document the new k8s Gateway API in this post.

I will consider that you are already familiar with kubernetes and will not go with the basics.

## Setup environment

### Minikube

Official documentation: https://minikube.sigs.k8s.io/docs/

**Current version:** `v1.32.0`

Once you have install it install we can start a new profile by running

```bash
minikube start -p test-gateway-api
```

Once the profile is started, you can validate it is up and running

```bash
# using minikube
$ minikube status -p test-gateway-api
test-gateway-api
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured


# using kubectl
$ kubectl get nodes
NAME               STATUS   ROLES           AGE    VERSION
test-gateway-api   Ready    control-plane   105s   v1.28.3
```

### Gateway API CRD

For those who know how kubernetes is built, CRDs allow to extented the k8s API capabilities.
We need to activate the api resources for Gateway API from the official github repository https://github.com/kubernetes-sigs/service-apis/.

To install it inside our cluster we can run the following command

```bash
kubectl apply -k "github.com/kubernetes-sigs/service-apis/config/crd?ref=v1.0.0"
```

We can confirm the required resources are present

```bash
$ kubectl api-resources
...
gatewayclasses                    gc           gateway.networking.k8s.io/v1     false        GatewayClass
gateways                          gtw          gateway.networking.k8s.io/v1     true         Gateway
httproutes                                     gateway.networking.k8s.io/v1     true         HTTPRoute
...
```

Now we have the resources to use the Kubernetes Gateway API

## Setup Kubernetes Gateway API

Based [official documentation](https://kubernetes.io/docs/concepts/services-networking/gateway/#resource-model), the gateway api is based on the 3 following resources:

- GatewayClass
- Gateway
- HttpRoute

`HttpRoute` ==> `Gateway` ==> `GatewayClass`

### Envoy Gateway

We will follow the current documentation version: `v0.6.0`

documentation: https://gateway.envoyproxy.io/v0.6.0/

```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm --version v0.0.0-latest -n envoy-gateway-system --create-namespace
```

We can validate that everything is installed and configured

```bash
$ kubectl get svc,deploy -n envoy-gateway-system
NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)               AGE
service/envoy-gateway                   ClusterIP   10.102.32.86     <none>        18000/TCP,18001/TCP   2d
service/envoy-gateway-metrics-service   ClusterIP   10.101.156.233   <none>        19001/TCP             2d

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/envoy-gateway   1/1     1            1           2d

```

All pieces are now in place and we can start to experiment.

## Application setup and tests

### Description

#### Resources

| namespace        | resource   | name             |
| ---------------- | ---------- | ---------------- |
| dev              | Service    | backend          |
| dev              | Deployment | backend          |
| dev              | HTTPRoute  | backend          |
| qa               | Service    | backend          |
| qa               | Deployment | backend          |
| qa               | HTTPRoute  | backend          |
| prod             | Service    | backend          |
| prod             | Deployment | backend          |
| prod             | HTTPRoute  | backend          |
| gateway-non-prod | Gateway    | gateway-non-prod |
| gateway-prod     | Gateway    | gateway-prod     |

#### Diagram

![Alt setup](./images/gateway-api.svg)

#### Installation

We can now install our services to run our tests

```bash
kubectl apply -k k8s/overlays/
namespace/dev created
namespace/gateway-non-prod created
namespace/gateway-prod created
namespace/prod created
namespace/qa created
serviceaccount/backend created
serviceaccount/backend created
serviceaccount/backend created
service/backend created
service/backend created
service/backend created
deployment.apps/backend created
deployment.apps/backend created
deployment.apps/backend created
gateway.gateway.networking.k8s.io/envoy-gateway created
gateway.gateway.networking.k8s.io/envoy-gateway created
gatewayclass.gateway.networking.k8s.io/envoy-gateway created
gatewayclass.gateway.networking.k8s.io/envoy-gateway unchanged
httproute.gateway.networking.k8s.io/backend created
httproute.gateway.networking.k8s.io/backend created
httproute.gateway.networking.k8s.io/backend created
```

#### Tests

First we can see that the creation of the gateway resources genereate new services in `envoy-gateway-system` namespace

```bash
$ kubectl get svc -n envoy-gateway-system
NAME                                            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)               AGE
envoy-gateway                                   ClusterIP      10.102.32.86     <none>        18000/TCP,18001/TCP   2d
envoy-gateway-metrics-service                   ClusterIP      10.101.156.233   <none>        19001/TCP             2d
envoy-gateway-non-prod-envoy-gateway-918f7df1   LoadBalancer   10.97.166.24     <pending>     80:31016/TCP          61s
envoy-gateway-prod-envoy-gateway-c1be5aa6       LoadBalancer   10.98.134.88     <pending>     80:31337/TCP          61s
```

Since we did not installed the tools to use a real load balancer, we will use port-forwarding

##### non-production

Let's start with non-production

```bash
service=`kubectl get svc -n envoy-gateway-system | grep gateway-non-prod | awk '{print $1}'`
kubectl -n envoy-gateway-system port-forward $(kubectl -n envoy-gateway-system get service $service --output=name) 8000:80
```

We can try to access our running service with curl on the opened host port

```bash
$ curl http://localhost:8000
$ curl http://localhost:8000 -v
*   Trying [::1]:8000...
* Connected to localhost (::1) port 8000
> GET / HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/8.4.0
> Accept: */*
>
< HTTP/1.1 404 Not Found
< date: Fri, 08 Dec 2023 02:10:25 GMT
< server: envoy
< content-length: 0
<
* Connection #0 to host localhost left intact
```

Taking a look to the HTTPRoute config in dev namespace

```bash
$ kubectl -n dev get httproute backend -o yaml
...
spec:
  hostnames:
  - dev.example.com
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: envoy-gateway
    namespace: gateway-non-prod
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: backend
      port: 3000
      weight: 1
    matches:
    - path:
        type: PathPrefix
        value: /
...
```

We can see that it listen hostname `dev.example.com`, let's modify our call

```bash
$ curl http://localhost:8000 -H 'Host: dev.example.com'
{
 "path": "/",
 "host": "dev.example.com",
 "method": "GET",
 "proto": "HTTP/1.1",
 "headers": {
  "Accept": [
   "*/*"
  ],
  "User-Agent": [
   "curl/8.4.0"
  ],
  "X-Envoy-Expected-Rq-Timeout-Ms": [
   "15000"
  ],
  "X-Envoy-Internal": [
   "true"
  ],
  "X-Forwarded-For": [
   "10.244.0.13"
  ],
  "X-Forwarded-Proto": [
   "http"
  ],
  "X-Request-Id": [
   "f14936bb-dac0-472c-a7f1-57625006831f"
  ]
 },
 "namespace": "dev",
 "ingress": "",
 "service": "",
 "pod": "backend-58d58f745-j79qb"
```

Bingo

## Sources

- Gateway API official documentation: https://kubernetes.io/docs/concepts/services-networking/gateway/
- Gateway official repository: https://github.com/kubernetes-sigs/gateway-api
- Minikube official documentation: https://minikube.sigs.k8s.io/docs/
- Envoy Gateway official documentation : https://doc.traefik.io/traefik/getting-started/install-traefik/
- Cross namespace: https://gateway-api.sigs.k8s.io/guides/multiple-ns/
