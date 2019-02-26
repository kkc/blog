---
title: Deploy Prometheus Operator With Thanos
catalog: true
date: 2019-02-10 21:53:12
subtitle: Solve HA and long term storage of Prometheus
header-img: univer.jpeg
tags:
  - Kubernetes
  - Prometheus
  - Thanos
---

# Preface

Prometheus is widely adopted as a standard monitoring tool with Kubernetes because it provides many useful features such as dynamic service discovery, powerful queries, and seamless alert notification integration. There are many applications and client libraries support Prometheus which makes the operation's life easier. Although things are going pretty well with prometheus, the original prometheus deployment is not able to easily achieve High Availablity and long term storage.

# Thanos comes to the rescue

![Thanos](./thanos.jpeg)

Thanos is developed by [improbable](https://github.com/improbable-eng) which can be integrated with prometheus transparently and solve HA and long term storage issues without hurting performance. The idea of Thanos is to run sidecar component of prometheus, therefore meaning that sidecar components can interact with prometheus to upload or query metrics. Also, prometheus operator supports thanos natively which make us easier to deploy our promtheus cluster along with thanos. This solution seems pretty elegant when you choose prometheus operator to provision prometheus cluster.

This article includes the following contents
  - How to deploy the prometheus operator on the kubernetes
  - How to deploy the thanos sidecar w/ prometheus.
  - Achieve HA: using thanos querier
  - Query historical data: thanos store
  - Reduce data size: thanos compactor

# Install Prometheus through Prometheus operator

There are tons of article introducing why we need to adopt prometheus-operator to provision prometheus. I recommend you read the following references[2] if you are not familiar with prometheus-operator.


## 1. Install Helm in your environment

- MacOS: `brew install kubernetes-helm`
- Linux: `sudo snap install helm`

## 2. Initialize helm and install tiller

```
$ helm init
```

## 3. Install coreos prometheus operator

Note that we are using `stable/prometheus-operator` because `coreos/prometheus-operator` helm is going to be deprecated. We later need to modify chart value to provision prometheus cluster along with thanos sidecar. To install a stable helm chart with custom value, you need to download `values.yaml` from [github repo](https://github.com/helm/charts/blob/master/stable/prometheus-operator/values.yaml).

In this example, we named our prometheus operator as `prom-op` and install it under `monitoring` namespace.

```
$ helm upgrade --install prom-op stable/prometheus-operator --namespace monitoring -f values.yaml
```

Use the following command to verify if prometheus-operator is provisioning successfully.
 
```
kubectl --namespace monitoring get pods -l "release=prom-op"
```

# Thanos Deployment

**NEED TO KNOW **
prometheus-operator should be greater than 0.28.0 to support Thanos 2.0

## Thanos Architecture

Official Architecture of Thanos
![arch](https://raw.githubusercontent.com/improbable-eng/thanos/master/docs/img/arch.jpg)

Our deployment steps
![arch](https://user-images.githubusercontent.com/17483589/45601152-096aba80-ba11-11e8-8d46-20f666583386.jpg)

According to the above picture, there are several components of thanos:
- Sidecar
- Querier
- Store
- Compactor

The deployment steps:
1. Prometheus should be deployed with thanos `Sidecar`.
2. Deploy Thanos `Querier` which is able to talks to prometheus `Sidecar` through gossip protocol.
3. Make sure Thanos `Sidecar` is able to upload prometheus metrics to the given S3 bucket.
4. Establish the Thanos `Store` for retrieving long term storage. 
5. Set up the `Compactor` to shrink historical data.

## Install Thanos sidecar

To install Thanos sidecar along with prometheus-operator, we should specify thanos sidecar in the chart value as following:

```
thanos:
    baseImage: improbable/thanos
    version: v0.2.1
    peers: thanos-peers.monitoring.svc:10900
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objstore-config
```

`objectStorageConfig` can be configured through configuration file `thanos.yaml`

```
type: s3
config:
  bucket: test-prometheus-thanos
  endpoint: s3.us-west-2.amazonaws.com
  encryptsse: true
```

Creating the kubernetes secret by applying following command

```
kubectl -n monitoring create secret generic thanos-objstore-config --from-file=thanos.yaml=/tmp/thanos-config.yaml
```

**Warn**: `endpoint` needs to be set in order to specify bucket located in which region.

## Verify Thanos Sidecar 

```
$ kubectl get po -n monitoring
```

```
kubectl describe po/prometheus-prom-op-prometheus-0 -n monitoring
```

If everything goes well, we could find out there is thanos-sidecar in the prometheus pod

```
  thanos-sidecar:
    Container ID:  docker://e52df9fda7b0c43eea297d273169cf33e4aa49780fd8d5192c23f497c78b2007
    Image:         improbable/thanos:v0.2.1
    Image ID:      docker-pullable://improbable/thanos@sha256:4ee0774316a5d57f78d243fe4afb10e9e889670d3facfdda70aae76f7165a16b
    Ports:         10902/TCP, 10901/TCP, 10900/TCP
    Host Ports:    0/TCP, 0/TCP, 0/TCP
    Args:
      sidecar
      --prometheus.url=http://127.0.0.1:9090
      --tsdb.path=/prometheus
      --cluster.address=[$(POD_IP)]:10900
      --grpc-address=[$(POD_IP)]:10901
      --cluster.peers=thanos-peers.monitoring.svc.cluster.local:10900
    State:          Running
      Started:      Fri, 01 Feb 2019 12:24:38 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      POD_IP:   (v1:status.podIP)
    Mounts:
      /prometheus from prometheus-prom-op-prometheus-db (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from prom-op-prometheus-token-7gvcp (ro)
```

and if you check the log of sidecar, you will see following messages.
```
kubectl log -f  po/prometheus-prom-op-prometheus-0 -n monitoring -c thanos-sidecar
```

```
level=info ts=2019-02-01T09:33:15.173007261Z caller=flags.go:90 msg="StoreAPI address that will be propagated through gossip" address=10.11.29.191:10901
level=info ts=2019-02-01T09:33:20.178094001Z caller=main.go:256 component=sidecar msg="disabled TLS, key and cert must be set to enable"
level=info ts=2019-02-01T09:33:20.178211091Z caller=factory.go:39 msg="loading bucket configuration"
level=info ts=2019-02-01T09:33:20.17855779Z caller=sidecar.go:280 msg="starting sidecar" peer=
level=info ts=2019-02-01T09:33:20.179145313Z caller=sidecar.go:220 component=sidecar msg="Listening for StoreAPI gRPC" address=[10.11.29.191]:10901
level=info ts=2019-02-01T09:33:20.179187469Z caller=main.go:308 msg="Listening for metrics" address=0.0.0.0:10902
level=info ts=2019-02-01T12:33:50.282222532Z caller=shipper.go:201 msg="upload new block" id=01D2MGSADK1860F4APSD7CFZ7C
```

## Install Thanos Querier

Thanos Querier Layer provides the ability to retrieve metrics from all prometheus instances at once. It's fully compatible with original prometheus PromQL and HTTP APIs so that it can be used along with Grafana.

Since there are too many yaml files, I put everything in my [github repo](https://github.com/kkc/prometheus-thanos)

```
$ cd thanos
$ kubectl apply -f querier-deployment.yaml
$ kubectl apply -f querier-service.yaml
$ kubectl apply -f querier-service-monitor.yaml
$ kubectl apply -f thanos-peers-svc.yaml
```

## Install Thanos Store

Thanos Store collaborates with `querier` for retrieving historical data from the given bucket. It will join the Thanos cluster on setup. 

```
$ kubectl apply -f thanos-store.yaml
```

## Install Thanos Compactor

Thanos Compactor will do downsampling for your all historical data. It's a really useful component which can reduce file size. Recommend everyone read this well explained [article](https://improbable.io/games/blog/thanos-prometheus-at-scale). 

```
$ kubectl apply -f thanos-compactor.yaml
$ kubectl apply -f thanos-compactor-service.yaml
$ kubectl apply -f thanos-compactor-service-monitor.yaml
```

# Troubleshooting

## Peering service didn't set up properly

you will see this kind of message of thanos component
```
level=error ts=2019-02-01T05:11:40.805153721Z caller=cluster.go:269 component=cluster msg="Refreshing memberlist" err="join peers thanos-peers.monitoring.svc.cluster.local:10900 : 1 error occurred:\n\t* Failed to resolve thanos-peers.monitoring.svc.cluster.local:10900: lookup thanos-peers.monitoring.svc.cluster.local on 172.20.0.10:53: no such host\n\n"
```

```
$ kubectl apply -f thanos-peers-svc.yaml
```

# References

1. https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md
2. https://sysdig.com/blog/kubernetes-monitoring-prometheus-operator-part3/
3. https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/alerting.md
4. [Thanos___Transforming_Prometheus_to_a_Global_Scale_in_a_Seven_Simple_Steps_(FOSDEM).pdf](https://fosdem.org/2019/schedule/event/thanos_transforming_prometheus_to_a_global_scale_in_a_seven_simple_steps/attachments/slides/3178/export/events/attachments/thanos_transforming_prometheus_to_a_global_scale_in_a_seven_simple_steps/slides/3178/Thanos___Transforming_Prometheus_to_a_Global_Scale_in_a_Seven_Simple_Steps_(FOSDEM).pdf)
