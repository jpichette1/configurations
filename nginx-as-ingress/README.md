# Setting up Nginx Ingress Controller as the Ingress for Istio / Aspen Mesh

## Overview

There have be some customer and customer pre-sales questions on whether it is feasible to a use Nginx Ingress Controller as an ingress rather than istio-ingressgateway.  The requirements range from customers already implementing and preferring to use Nginx as Ingress, to the addition of Istio's ingressgateway (Nginx Ingress / Istio Ingress / service mesh routing) creates additional overhead and latency.  The result is the customer wants to bypass Istio Ingress Gateway altogether.

Implementing Nginx Ingress Controller as an ingress for Istio / Aspen Mesh is achieved by adding an envoy sidecar proxy to Nginx pods and having Nginx redirect traffic to a kubernetes Ingress object by adding specific `annotations` to both the Nginx and kubernetes Ingress objects.  

This document outlines many of the steps necessary to implement this configuration.

## Architecture

![Nginx Ingress Controller as an Ingress to Istio / Aspen Mesh -  Architecture](link-to-image)

## Prerequisites

There are several requirements you'll need to have in place before you begin:

- An existing kubernetes cluster that allows Nginx to assign an external-IP (such as EKS or GKE)
- Aspen Mesh v1.6 or higher, is installed and healthy.
- `kubectl`
- capability to assign an A record DNS name for the Nginx Ingress Controller
- Latest Ngnix Ingress Controller 0.29 or greater
- Istio's bookinfo sample application (or some demo app with automatic sidecar injection enabled)

## Preparation

Create a namespace for the Nginx Ingress Controller and enable automatic Istio sidecar injection

```
kubectl create namespace nginx-ingress
kubectl label namespace nginx-ingress istio-injection=enabled
```

## Install Nginx Ingress Controller

Install the latest version of Nginx Ingress Controller.

```
helm install nginx-ingress stable/nginx-ingress --namespace nginx-ingress --set rbac.create=true --set controller.publishService.enabled=true
```

### Add Nginx Ingress Controller annotations

You will need to add additional annotations to the Nginx-ingress deployments to instruct Nginx to re-direct traffic through the envoy sidecar into the Istio service mesh.

```
kubectl -n nginx-ingress edit deployment nginx-ingress-controller
```

In the `nginx-ingress-controller` deployment, add the following annotation:

```
template:
    metadata:
      annotations:
        traffic.sidecar.istio.io/includeInboundPorts: ""
```

```
kubectl -n nginx-ingress edit deployment nginx-ingress-default-backend -o yaml
```

Because Nginx Ingress Controller is installed into a sidecar enabled namespace, the Liveness probe for `nginx-ingress-default-backend` pod will be blocked by Istio mTLS.  To allow the Liveness probe to successfully complete checks, the probe can be redirected to pass through the pilot agent by adding the `rewriteAppHTTPProbers: "true"` annotation.

In the `nginx-ingress-default-backend` add the following annotations.

```
template:
    metadata:
      annotations:
        traffic.sidecar.istio.io/includeInboundPorts: ""
        sidecar.istio.io/rewriteAppHTTPProbers: "true"
```

Create a DNS A record for the Nginx-ingress-controller loadbalancer.  Use the External-IP of the nginx-ingress-controller service.

```
kubectl -n nginx-ingress get service
NAME                                    TYPE            CLUSTER-IP      EXTERNAL-IP     PORT(S)         
nginx-ingress-controller                LoadBalancer    10.7.246.185    35.194.12.35          80:30444/TCP,443:31779/TCP 
nginx-ingress-default-backend           ClusterIP       10.7.246.244    <none>          80/TCP                      
```

## Create an Ingress policy to route traffic

As an example, have the Nginx-ingress-controller route to bookinfo's productpage.  To do this, create an Ingress object with nginx-ingress specific annotations and add the specific nginx-ingress-controller hostname as a rule.

```
kubectl apply -f productpage-nginx-ingress.yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/service-upstream: "true"
    nginx.ingress.kubernetes.io/upstream-vhost: productpage.default.svc.cluster.local
  name: productpage-ingress
  namespace: default
spec:
  rules:
  - host: productpage.nginx.rocketj.io
  - http:
      paths:
      - backend:
          serviceName: productpage
          servicePort: 9080
        path: /productpage

```

Validate sidecars are receiving the proper configuration details and that the services are in-sync.

```
$ istioctl proxy-status
NAME                                        CDS     LDS     EDS     RDS     PILOT           VERSION
productpage-xxxxx.default                   SYNCED  SYNCED  SYNCED  SYNCED  istio-pilot-xxx 1.6.x
nginx-ingress-controller-xxx.nginx-ingress  SYNCED  SYNCED  SYNCED  SYNCED  istio-pilot-xxx 1.6.x

```
