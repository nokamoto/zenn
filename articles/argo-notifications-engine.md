---
title: "Argo Notifications Engine を動かしてみる"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["argo"]
published: true
---

KubeCon + CloudNativeCon North America 2021 の [Easy notifications for Kubernetes](https://kccncna2021.sched.com/event/lV5p/easy-notifications-for-kubernetes-alexander-matyushentsev-intuit-remington-breeze-akuity) で紹介されていた [Argo Notifications Engine](https://github.com/argoproj/notifications-engine) を試しに動かしてみる。

# 対象のリポジトリ
https://github.com/nokamoto/argo-notifications-engine-for

# アプリケーションの実装
[examples](https://github.com/argoproj/notifications-engine/tree/master/examples/certmanager) を参考にアプリケーションを実装する。

https://github.com/nokamoto/argo-notifications-engine-for/blob/main/cmd/pods/main.go

```go
// Create notifications controller that handles Kubernetes resources processing
podClient := dynamic.NewForConfigOrDie(restConfig).Resource(schema.GroupVersionResource{
    Group: "", Version: "v1", Resource: "pods",
})
```

このアプリケーションでは Pod の状態を監視するように記述してみた。

# アプリケーションのデプロイ
どのように動かせば良いのか説明を見つけられなかったので動かしつつ必要な設定を手探りで入れた。

https://github.com/nokamoto/argo-notifications-engine-for/blob/main/deployments/tilt.yaml

このアプリケーションは kubeconifg を読み込んで Kubernetes のリソースにアクセスするため、実行する ServiceAccount にいくつかの権限を付けて動かしている。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-resources-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "watch", "patch"]
```

アプリケーションの記述で今回は Pod の状態を監視するようにしたため、実行にはクラスタレベルの Pod に関する権限を要求するようだった。
`"patch"` の権限は後述する通知のために利用されているようだ。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: namespace-resources-reader
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["list", "get", "watch"]
```

アプリケーションの設定を読み込むために、実行には namespace レベルの Secret/ConfigMap に関する権限を要求するようだった。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pod-notifications-cm
data:
  trigger.on-pod-ready: |
    - when: pod.status.phase in ['Running']
      send: [pod-ready]
  template.pod-ready: |
    webhook:
      webhook:
        method: GET
        path: /
        body: |
          Pod {{.pod.metadata.name}} is ready!
  service.webhook.webhook: |
    url: http://webhook.default:8080/webhook
```

アプリケーションの設定については [docs](https://github.com/argoproj/notifications-engine/tree/master/docs) に説明がある。
今回の設定では検証のため `pod.status.phase` が `Running` になったことをトリガーに、 webhook に対して HTTP リクエストを送るような設定を入れている。

```
annotations:
    notifications.argoproj.io/subscribe.on-pod-ready.webhook: ""
```

監視したい対象の Pod には必要なアノテーションをつけることで、 アプリケーションの監視対象となる。

# アプリケーションの実行

```
time="2022-02-17T06:16:38Z" level=info msg="Trigger on-pod-ready result: [{[0].jS5fTrIQi8Q4Q5uceLlcAF3t4Go  [pod-ready] true}]" resource=default/webhook-5bd87bd4f-w7m46
time="2022-02-17T06:16:38Z" level=info msg="Sending notification about condition 'on-pod-ready.[0].jS5fTrIQi8Q4Q5uceLlcAF3t4Go' to '{webhook }'" resource=default/webhook-5bd87bd4f-w7m46
```

アプリケーションを動かすとアノテーションが付いた Pod を検知すると通知を送る。

```
2022/02/17 06:16:38 [172.17.0.1:40962 GET /webhook/] body=Pod webhook-5bd87bd4f-w7m46 is ready!
```

Webhook として設定した送信先ではテンプレートで設定した通知が届いていることが確認できる。

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-pod-ready.webhook: ""
    notified.notifications.argoproj.io: '{"on-pod-ready:[0].jS5fTrIQi8Q4Q5uceLlcAF3t4Go:webhook:":1645078598}'
```

この時、検知された Pod の状態を確認すると `notified.notifications.argoproj.io` アノテーションが追加されていることが分かる。
この `patch` により、通知の重複が防がれているのだと考えられる[^1]。

[^1]: `patch` を設定しなかった場合、通知が送られ続けるような挙動となった。

```
time="2022-02-17T06:16:38Z" level=info msg="Trigger on-pod-ready result: [{[0].jS5fTrIQi8Q4Q5uceLlcAF3t4Go  [pod-ready] true}]" resource=default/webhook-5bd87bd4f-w7m46
time="2022-02-17T06:16:38Z" level=info msg="Notification about condition 'on-pod-ready.[0].jS5fTrIQi8Q4Q5uceLlcAF3t4Go' already sent to '{webhook }'" resource=default/webhook-5bd87bd4f-w7m46
```

二回目以降、同じ状態の検知が行われた場合は既に送られたという出力になり、 Webhook への通知が行われることがなくなる。

# まとめ

今回は Pod を監視するようなアプリケーションを書いたが、 `trigger` で記述した任意の Kubernetes リソースの状態に対して `service` で記述した任意の通知を送ることが出来そうだった。
ドキュメントが見つけられなかったので設定関連に難しさを感じたが、仕組み的には何かに使えそうなので使い方を考え中。
