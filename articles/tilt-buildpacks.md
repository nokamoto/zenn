---
title: "Buildpacks を Tilt で導入する"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tilt", "buildpacks"]
published: true
---

[tilt.dev](https://docs.tilt.dev/) で buildpacks の使い方のドキュメントが見つけられなかったので記録しておく。

# 対象のプルリクエスト

https://github.com/nokamoto/sandbox/pull/75

- Tilt v0.22.15, built 2021-10-29
- Kind v0.11.1 go1.16.4 linux/amd64

devcontainer 内での実行で確認。

# Tilt + Buildpacks のドキュメント

公式で検索すると blog が見つかる。

https://blog.tilt.dev/2020/05/04/april-2020-commit-of-the-month

blog からの参照で extension の README が見つかる。

https://github.com/tilt-dev/tilt-extensions/blob/master/pack/README.md

以上を参考に `Tiltfile` を記述した。

# Tiltfile

```starlark
load('ext://pack', 'pack')

pack('example', builder = 'gcr.io/buildpacks/builder:v1')

k8s_yaml('deployments/example.yaml')
```

`Tiltfile` に `pack` と `k8s_yaml` を記述する。

この `pack` の記述で `pack build example:tilt-build-pack-caching --path . --builder gcr.io/buildpacks/builder:v1 --pull-policy if-not-present  --env BP_LIVE_RELOAD_ENABLED=true && docker tag example:tilt-build-pack-caching $EXPECTED_REF` のようなコマンドが実行されていた。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: example
  name: example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
      - image: example
        name: example
```

`pack` の第一引数と `spec.containers.image` のイメージ名が一致していれば動く。
