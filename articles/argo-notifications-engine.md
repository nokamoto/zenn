---
title: "Argo Notifications Engine を動かしてみる"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["argo"]
published: true
---

KubeCon + CloudNativeCon North America 2021 で見た [Easy notifications for Kubernetes](https://kccncna2021.sched.com/event/lV5p/easy-notifications-for-kubernetes-alexander-matyushentsev-intuit-remington-breeze-akuity) で紹介されていた [Argo Notifications Engine](https://github.com/argoproj/notifications-engine) を試しに動かしてみる。

# 対象のリポジトリ
https://github.com/nokamoto/argo-notifications-engine-for

# アプリケーションの実装
[examples](https://github.com/argoproj/notifications-engine/tree/master/examples/certmanager) を参考にアプリケーションを実装する。

https://github.com/nokamoto/argo-notifications-engine-for/blob/main/cmd/pods/main.go

```
// Create notifications controller that handles Kubernetes resources processing
podClient := dynamic.NewForConfigOrDie(restConfig).Resource(schema.GroupVersionResource{
    Group: "", Version: "v1", Resource: "pods",
})
```

このアプリケーションでは Pod の状態を監視するように記述している。

# アプリケーションのデプロイ
どのように動かせば良いのか説明を見つけられなかったので動かしつつ必要な設定を手探りで入れた。

https://github.com/nokamoto/argo-notifications-engine-for/blob/main/deployments/tilt.yaml

このアプリケーションは kubeconifg を読み込んで Kubernetes のリソースにアクセスするため、実行する ServiceAccount にいくつかの権限を付けて動かしている。

```
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

```
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

```
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
