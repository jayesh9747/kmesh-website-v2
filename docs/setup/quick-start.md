---
title: Quick Start
description: This guide lets you quickly install Kmesh.
sidebar_position: 1
---

# Quick Start

This guide lets you quickly install Kmesh.

## Preparation

Kmesh needs to run on a Kubernetes cluster. Kubernetes 1.26+ is currently supported. We recommend using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) to quickly provide a Kubernetes cluster (We provide a [document](https://kmesh.net/en/docs/setup/develop_with_kind/) for developing and deploying Kmesh using kind). Of course, you can also use [minikube](https://minikube.sigs.k8s.io/docs/) and other ways to create Kubernetes clusters.

Currently, Kmesh makes use of [istio](https://istio.io/) as its control plane. Before installing Kmesh, please install the Istio control plane. We recommend installing istio ambient mode because Kmesh `dual-engine` mode needs it. For details, see [ambient mode istio](https://istio.io/latest/docs/ops/ambient/getting-started/).

You can view the results of istio installation using the following command:

```sh
kubectl get po -n istio-system
NAME                      READY   STATUS    RESTARTS   AGE
istio-cni-node-xbc85      1/1     Running   0          18h
istiod-5659cfbd55-9s92d   1/1     Running   0          18h
ztunnel-4jlvv             1/1     Running   0          18h
```

:::note
To use waypoint, you need to install the Kubernetes Gateway API CRDs, which don’t come installed by default on most Kubernetes clusters:
:::

```sh
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref=444631bfe06f3bcca5d0eadf1857eac1d369421d" | kubectl apply -f -; }
```

## Only install Istiod

Installing ambient mode istio by the above steps will install additional istio components.

The process of installing only `istiod` as the control plane for Kmesh is provided next.

### Install Istio CRDs

```sh
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

To install the chart with the release name `istio-base`:

```sh
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system
```

### Install Istiod

To install the chart with the release name **Istiod**:

```sh
helm install istiod istio/istiod --namespace istio-system --set pilot.env.PILOT_ENABLE_AMBIENT=true
```

:::note
Must set `pilot.env.PILOT_ENABLE_AMBIENT=true`. Otherwise, Kmesh will not be able to establish gRPC links with istiod!
:::

After installing istiod, it's time to install Kubernetes Gateway API CRDs.

```sh
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref=444631bfe06f3bcca5d0eadf1857eac1d369421d" | kubectl apply -f -; }
```

## Install Kmesh

We offer several ways to install Kmesh.

### Install from Helm

```sh
helm install kmesh ./deploy/charts/kmesh-helm -n kmesh-system --create-namespace
```

### Install from YAML

```sh
kubectl create namespace kmesh-system
kubectl apply -f ./deploy/yaml/
```

You can confirm the status of Kmesh with the following command:

```sh
kubectl get pod -n kmesh-system
```

## Deploy the Sample Applications

Kmesh can manage pods in a namespace with a label `istio.io/dataplane-mode=Kmesh`, and the pod should not have the `istio.io/dataplane-mode=none` label.

```sh
kubectl label namespace default istio.io/dataplane-mode=Kmesh
```

Deploy sleep and httpbin:

```sh
kubectl apply -f ./samples/httpbin/httpbin.yaml
kubectl apply -f ./samples/sleep/sleep.yaml
```

Check application status:

```sh
kubectl get pod
```

To confirm if a pod is managed by Kmesh:

```sh
kubectl describe po httpbin-65975d4c6f-96kgw | grep Annotations
```

## Test Service Access

Test that the applications can still communicate successfully.

```sh
kubectl exec sleep-7656cf8794-xjndm -c sleep -- curl -IsS "http://httpbin:8000/status/200"
```

## Clean Up

To stop Kmesh from managing the application:

```sh
kubectl label namespace default istio.io/dataplane-mode-
kubectl delete pod httpbin-65975d4c6f-96kgw sleep-7656cf8794-8tp9n
```

To uninstall Kmesh:

### If installed using Helm

```sh
helm uninstall kmesh -n kmesh-system
kubectl delete ns kmesh-system
```

### If installed using YAML

```sh
kubectl delete -f ./deploy/yaml/
```

To remove the sleep and httpbin applications:

```sh
kubectl delete -f samples/httpbin/httpbin.yaml
kubectl delete -f samples/sleep/sleep.yaml
```

To remove the Gateway API CRDs:

```sh
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref=444631bfe06f3bcca5d0eadf1857eac1d369421d" | kubectl delete -f -
```
