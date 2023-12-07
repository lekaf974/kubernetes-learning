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

All pieces are now in place and we can start to experiment.

## Application setup and tests

### Description

#### Resources

| namespace | resource   | name           |
| --------- | ---------- | -------------- |
| dev       | Service    | backend        |
| dev       | Deployment | backend        |
| dev       | HTTPRoute  | backend        |
| qa        | Service    | backend        |
| qa        | Deployment | backend        |
| qa        | HTTPRoute  | backend        |
| prod      | Service    | backend        |
| prod      | Deployment | backend        |
| prod      | HTTPRoute  | backend        |
| gateway   | Gateway    | non-production |
| gateway   | Gateway    | production     |

#### Diagram

![Alt setup](./images/gateway-api.svg)

#### Installation

```bash
kubectl apply -k k8s/overlays
```

## Sources

- Gateway API official documentation: https://kubernetes.io/docs/concepts/services-networking/gateway/
- Gateway official repository: https://github.com/kubernetes-sigs/gateway-api
- Minikube official documentation: https://minikube.sigs.k8s.io/docs/
- Envoy Gateway official documentation : https://doc.traefik.io/traefik/getting-started/install-traefik/
- Cross namespace: https://gateway-api.sigs.k8s.io/guides/multiple-ns/
