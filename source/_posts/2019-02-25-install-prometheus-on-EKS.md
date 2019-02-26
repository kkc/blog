---
title: 利用 Helm 在 EKS 上安裝 Prometheus 
catalog: true
date: 2019-02-25 16:39:36
subtitle: 一些踩雷經驗
header-img: fire.jpeg
tags:
  - prometheus
  - EKS
  - AWS
  - monitoring
  - kubernetes
  - helm
---

# Preface

最近把玩了 EKS 一陣子，基本上 EKS 就是 AWS 提供的 Managed Kubernetes，主要是幫你管理 Kubernetes 的 master node，我們只需要管理 worker node 就好了，所以很多的服務還是可以用原本的 helm chart 裝起來，這篇文章會介紹怎麼在 EKS 上面利用 helm 安裝 Prometheus 相關的套件，還有一些簡單的設定。

這篇文章會包含以下內容
- 利用 helm 安裝 Prometheus-operator 再透過 Operator 去部署 prometheus & alertmanager
- 如何設定 helm value 去避免一些 EKS 上面的錯誤問題
- Troubleshooting 的一些 tips

# 利用 helm 安裝 prometheus

因為 `coreos/prometheus-operator` 的 helm chart 已經被 deprecated 掉了，所以我們這邊會使用 `stable/prometheus-operator` 去做安裝，而這包 chart 其實有包含蠻多 components 像是 `prometheus` & `alertmanager` ，還會幫你裝好 prometheus 需要監控用的 `node-exporter` 等等東西，所以非常大一包，很建議大家裝好後，可以回過頭來看看到底被安裝了哪些東西。

## 確認 stable/prometheus-operator 版本
```
$ helm search -l stable/prometheus-operator
```

可以看到目前最新的 Chart 版本是 `4.0.0`
```
NAME                             CHART VERSION   APP VERSION     DESCRIPTION
stable/prometheus-operator      4.0.0           0.29.0          Provides easy monitoring definitions for Kubernetes servi...
stable/prometheus-operator      3.0.0           0.29.0          Provides easy monitoring definitions for Kubernetes servi...
stable/prometheus-operator      2.6.0           0.27.0          Provides easy monitoring definitions for Kubernetes servi...
```

安裝，這邊我們把安裝的名字取作 `prom-op`
```
$ helm install --name prom-op --namespace monitoring stable/prometheus-operator
```

透過以下的指令可以得知安裝了些什麼東西
```
$ kubectl --namespace monitoring get pods
```

```
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prom-op-prometheus-operato-alertmanager-0   2/2     Running   0          1m
prom-op-grafana-5c59ddfb9d-zqfqt                         2/2     Running   0          2m
prom-op-kube-state-metrics-76786cc9b4-8q4bj              1/1     Running   0          2m
prom-op-prometheus-node-exporter-6jclc                   1/1     Running   0          2m
prom-op-prometheus-node-exporter-bxr49                   1/1     Running   0          2m
prom-op-prometheus-node-exporter-mxtht                   1/1     Running   0          2m
prom-op-prometheus-node-exporter-xd54m                   1/1     Running   0          2m
prom-op-prometheus-operato-operator-6cbf5d5cfd-z6fz4     1/1     Running   0          2m
prometheus-prom-op-prometheus-operato-prometheus-0       3/3     Running   1          1m
```

因為我這台 k8s cluster 有起了 4 個 node，所以會安裝 4 個 node operator，然後還會安裝 prometheus-operator, alertmanager, grafana 和 kube-state-metrics。

## Customizing the Chart

透過 port forward 讀取 localhost:9090 可以看到 prometheus 裡面的資訊
```
$ kubectl port-forward svc/prom-op-prometheus-operato-prometheus -n monitoring 9090 
```

其中我們會看到以下這些錯誤
![prometheus-error](./prometheus-error.png)
![prometheus-error-2](./prometheus-error-2.png)

因為我們無法監控到 EKS 的 master node，所以關於 master 上面的 services 像是 etcd, kube-apiserver, controller-manager, kube-schedule 都會在 prometheus 中發生錯誤，這也是為什麼我們需要客製化我們的 chart file。
```
$ cp https://raw.githubusercontent.com/helm/charts/master/stable/prometheus-operator/values.yaml values.yaml
```

修改完後可以使用以下指令去覆寫
```
helm upgrade --install prom-op stable/prometheus-operator --namespace monitoring -f values.yaml
```

這邊筆記下我有更改的部分，master 上面的 services 像是 etcd, kube-apiserver, controller-manager, kube-schedule 等等的 monitoring 機制需要被關閉
```
kubeApiServer
  enabled: false

kubeControllerManager
  enabled: false

kubeEtcd
  enabled: false

kubeScheduler
  enabled: false
```

kubelet 的話根據這個 [issue](https://github.com/coreos/prometheus-operator/issues/926)，在 EKS 上面使用的話，我們需要把 https 的部分 enable 起來

```
kubelet:
  enabled: true
  namespace: kube-system

  serviceMonitor:
    https: true
```

EKS 上面的 coreDns 的 label 有點怪，還是用 k8s-app:kube-dns 而不是 coredns
```
coreDns:
  enabled: true
  service:
    port: 9153
    targetPort: 9153
    selector:
      k8s-app: kube-dns
```

還有一些 resource 的部分記得要調整下

```
resources:
  requests:
    memory: 400Mi
```

# 設定 addtional scrape config

Prometheus 除了可以用來 monitor Kubernetes 內部的 service 外，其實也有提供一些方法去 scrape 外面的 service，像是有一些程式跑在既有的 EC2 上面，我們可以透過相對應的 EC2 service discovery 的方法去拉取資料，要達成相關的任務，則需要去設定 addtional config。

方法很簡單，需要先在 chart 的 value 中把原本的 additionalScrapeConfigs

```
additionalScrapeConfigs: []
```
改寫為需要另外掛上去的 config

```
additionalScrapeConfigs:
  - job_name: placeholder
    metrics_path: /probe
    params:
    module: [http_2xx]
    static_configs:
      - targets:
        - https://sentry.umbocv.com/_health/?full
```

但是這種做法需要一直更改 helm chart 的 value，而這邊也提供另外一種方法可以直接更改 config，讓 prometheus config reloader 去讀取，使用
```
kubectl get secret -n monitoring
```
會看到有

```
NAME                                           TYPE                                  DATA   AGE
prom-op-prometheus-scrape-confg                Opaque                                1      30s
```

我們可以透過直接更改這個 secret 的內容而改動 addtional-scrape-config，而以下這個 addtional-scrape-configs.yaml 以上面的例子會長成這樣

```
  - job_name: placeholder
    metrics_path: /probe
    params:
    module: [http_2xx]
    static_configs:
      - targets:
        - https://sentry.umbocv.com/_health/?full
```

接著透過這行指令把這個 `addtional-scrape-configs.yaml` 轉成 k8s 認得的 secret yaml，在 apply 上去
```
$ kubectl create secret generic prom-op-prometheus-scrape-confg --from-file=additional-scrape-configs.yaml --dry-run -oyaml > prometheus-additional-scrape-configs.yaml
$ kubectl apply -f prometheus-additional-scrape-configs.yaml -n monitoring
```

# 設定 alert manager template

在使用完 prometheus-operator 的 helm 部署完後，其實可以從 UI 中的 status -> rules 中看到許多內建好的 prometheus 的 rule，而如果想要把這個警告發到 slack 上面還需要設定 alertmanager 的 route config，而內建的 config 其實沒做任何事情，都是導到 null 而已

```
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'null'
      routes:
      - match:
          alertname: Watchdog
        receiver: 'null'
    receivers:
    - name: 'null'
```

而這邊我們可以參考 Monza 的 [alertmanager slack template](https://gist.github.com/milesbxf/e2744fc90e9c41b47aa47925f8ff6512) ，這個 template 的好處就是可以幫 alert 都合併為一個發出來，然後也有吃內建的 rule 的 format，舉個例子像下面的這個 rule，裡面用到的 labels 是 `serverity: critical`，然後 annotations 裡面是 `message` & `runbook_url`

```
alert: KubeAPIDown
expr: absent(up{job="apiserver"}
  == 1)
for: 15m
labels:
  severity: critical
annotations:
  message: KubeAPI has disappeared from Prometheus target discovery.
  runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapidown
```

而透過 Monza 的 template 我們可以先設定 alertmanager 的 endpoint

```
receivers:
###################################################
## Slack Receivers
- name: slack-code-owners
  slack_configs:
  - channel: '#{{- template "slack.monzo.code_owner_channel" . -}}'
    send_resolved: true
    title: '{{ template "slack.monzo.title" . }}'
    icon_emoji: '{{ template "slack.monzo.icon_emoji" . }}'
    color: '{{ template "slack.monzo.color" . }}'
    text: '{{ template "slack.monzo.text" . }}'
    actions:
    - type: button
      text: 'Runbook :green_book:'
      url: '{{ (index .Alerts 0).Annotations.runbook_url }}'
    - type: button
      text: 'Query :mag:'
      url: '{{ (index .Alerts 0).GeneratorURL }}'
    - type: button
      text: 'Dashboard :grafana:'
      url: '{{ (index .Alerts 0).Annotations.dashboard }}'
    - type: button
      text: 'Silence :no_bell:'
      url: '{{ template "__alert_silence_link" . }}'
    - type: button
      text: '{{ template "slack.monzo.link_button_text" . }}'
      url: '{{ .CommonAnnotations.link_url }}'
```

在透過定義好的 template 中，我們可以看到已經有確認收到的警告是 `.Annotations.message` 會被顯示出來，這樣一來就可以把相關的 rule alert 打到 slack 上了。

```
# This builds the silence URL.  We exclude the alertname in the range
# to avoid the issue of having trailing comma separator (%2C) at the end
# of the generated URL
{{ define "__alert_silence_link" -}}
    {{ .ExternalURL }}/#/silences/new?filter=%7B
    {{- range .CommonLabels.SortedPairs -}}
        {{- if ne .Name "alertname" -}}
            {{- .Name }}%3D"{{- .Value -}}"%2C%20
        {{- end -}}
    {{- end -}}
    alertname%3D"{{ .CommonLabels.alertname }}"%7D
{{- end }}

{{ define "__alert_severity_prefix" -}}
    {{ if ne .Status "firing" -}}
    :lgtm:
    {{- else if eq .Labels.severity "critical" -}}
    :fire:
    {{- else if eq .Labels.severity "warning" -}}
    :warning:
    {{- else -}}
    :question:
    {{- end }}
{{- end }}

{{ define "__alert_severity_prefix_title" -}}
    {{ if ne .Status "firing" -}}
    :lgtm:
    {{- else if eq .CommonLabels.severity "critical" -}}
    :fire:
    {{- else if eq .CommonLabels.severity "warning" -}}
    :warning:
    {{- else if eq .CommonLabels.severity "info" -}}
    :information_source:
    {{- else -}}
    :question:
    {{- end }}
{{- end }}


{{/* First line of Slack alerts */}}
{{ define "slack.monzo.title" -}}
    [{{ .Status | toUpper -}}
    {{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{- end -}}
    ] {{ template "__alert_severity_prefix_title" . }} {{ .CommonLabels.alertname }}
{{- end }}


{{/* Color of Slack attachment (appears as line next to alert )*/}}
{{ define "slack.monzo.color" -}}
    {{ if eq .Status "firing" -}}
        {{ if eq .CommonLabels.severity "warning" -}}
            warning
        {{- else if eq .CommonLabels.severity "critical" -}}
            danger
        {{- else -}}
            #439FE0
        {{- end -}}
    {{ else -}}
    good
    {{- end }}
{{- end }}


{{/* Emoji to display as user icon (custom emoji supported!) */}}
{{ define "slack.monzo.icon_emoji" }}:prometheus:{{ end }}

{{/* The test to display in the alert */}}
{{ define "slack.monzo.text" -}}
    {{ range .Alerts }}
        {{- if .Annotations.message }}
            {{ .Annotations.message }}
        {{- end }}
        {{- if .Annotations.description }}
            {{ .Annotations.description }}
        {{- end }}
    {{- end }}
{{- end }}



{{- /* If none of the below matches, send to #monitoring-no-owner, and we 
can then assign the expected code_owner to the alert or map the code_owner
to the correct channel */ -}}
{{ define "__get_channel_for_code_owner" -}}
    {{- if eq . "platform-team" -}}
        platform-alerts
    {{- else if eq . "security-team" -}}
        security-alerts
    {{- else -}}
        monitoring-no-owner
    {{- end -}}
{{- end }}

{{- /* Select the channel based on the code_owner. We only expect to get
into this template function if the code_owners label is present on an alert.
This is to defend against us accidentally breaking the routing logic. */ -}}
{{ define "slack.monzo.code_owner_channel" -}}
    {{- if .CommonLabels.code_owner }}
        {{ template "__get_channel_for_code_owner" .CommonLabels.code_owner }}
    {{- else -}}
        monitoring
    {{- end }}
{{- end }}

{{ define "slack.monzo.link_button_text" -}}
    {{- if .CommonAnnotations.link_text -}}
        {{- .CommonAnnotations.link_text -}}
    {{- else -}}
        Link
    {{- end }} :link:
{{- end }}
```

這邊還有一個很重要的步驟，讓我卡了蠻久的，其實 template 也是一樣定義在 prometheus-operator 的 helm chart value.yaml 裡面，在定義完 template 後，一定要加上 
```
templates:
    - '/etc/alertmanager/config/*.tmpl'
```

大概的範例長得像這樣

```
  config
    global:
      resolve_timeout: 5m
    ...略

  templates:
    - '/etc/alertmanager/config/*.tmpl'
     
  templateFiles:
      template_monzo.tmpl: |-

         {{ define "__alert_silence_link" -}}
            {{ .ExternalURL }}/#/silences/new?filter=%7B
            {{- range .CommonLabels.SortedPairs -}}
                {{- if ne .Name "alertname" -}}
                    {{- .Name }}%3D"{{- .Value -}}"%2C%20
                {{- end -}}
            {{- end -}}
            alertname%3D"{{ .CommonLabels.alertname }}"%7D
        {{- end }}
        ...略
```

# Troubleshoot

1. 如果一直沒收到 alert 的話，有可能是 alertmanager 的 template 寫錯，可以透過 `kubectl logs -f po/<alertmanager_pod_name> -n monitoring -c alertmanager` 去確認下是不是有產生一些 error log。

2. 想要確認 alertmanager template 的語法的話，可以使用下面這個 script 去測試，主要是從這個 [gist](https://gist.github.com/cherti/61ec48deaaab7d288c9fcf17e700853a) 看來的，這樣就可以邊改 template 邊驗證，不用真的去產生一些錯誤條件出來。

   ```
#!/bin/bash

name=$RANDOM
url='http://localhost:9093/api/v1/alerts'

echo "firing up alert $name" 

# change url o
curl -XPOST $url -d "[{ 
	\"status\": \"firing\",
	\"labels\": {
		\"alertname\": \"$name\",
		\"service\": \"my-service\",
		\"severity\":\"warning\",
		\"instance\": \"$name.example.net\"
	},
	\"annotations\": {
		\"summary\": \"High latency is high!\"
	},
	\"generatorURL\": \"http://prometheus.int.example.net/<generating_expression>\"
}]"

echo ""

echo "press enter to resolve alert"
read

echo "sending resolve"
curl -XPOST $url -d "[{ 
	\"status\": \"resolved\",
	\"labels\": {
		\"alertname\": \"$name\",
		\"service\": \"my-service\",
		\"severity\":\"warning\",
		\"instance\": \"$name.example.net\"
	},
	\"annotations\": {
		\"summary\": \"High latency is high!\"
	},
	\"generatorURL\": \"http://prometheus.int.example.net/<generating_expression>\"
}]"
  ```

或是用

  ```
#!/bin/bash

alerts='[
  {
    "labels": {
       "alertname": "instance_down",
       "instance": "example1"
     },
     "annotations": {
        "info": "The instance example1 is down",
        "summary": "instance example1 is down"
      }
  }
]'

URL="https://alertmanager.mydomain.com"

curl -XPOST -d"$alerts" $URL/api/v1/alerts
   ```

3. 可以使用看看是否自己的 secret 內容是正確的

   ```
   kubectl get secret -n monitoring alertmanager-prom-op-alertmanager -o go-template='{{ index .data "alertmanager.yaml" }}' | base64
   ```


# 完整移除 prometheus-operator

```
$ helm delete --purge <name>
$ kubectl delete crd prometheuses.monitoring.coreos.com
$ kubectl delete crd prometheusrules.monitoring.coreos.com
$ kubectl delete crd servicemonitors.monitoring.coreos.com
$ kubectl delete crd alertmanagers.monitoring.coreos.com
```

# 後記

原本使用 prometheus-operator 其實還有個雷就是 servicemonitor 需要打上 `release: <deploy_name>`，這樣 operator 才真的會去吃這個 service monitor，但是隨著 4.0.0 的更新也把這個惱人的東西修掉了，所以建議大家常常去看下到底更新了什麼，其實 prometheus & alertmanager 的版本也是一直推進很快的，而接下來有想到什麼更多的內容，還會繼續更新這篇。

# Reference
1. https://github.com/helm/charts/tree/master/stable/prometheus-operator
2. https://github.com/prometheus/alertmanager/issues/437

