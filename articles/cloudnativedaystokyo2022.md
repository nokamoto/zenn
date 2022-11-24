---
title: "CloudNative Days Tokyo 2022 の感想"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cndt2022"]
published: true
---

Cloud Native Days Tokyo 2022 にオフライン参加した。

日程: 2022/11/21 - 2022/11/22
会場: 有明セントラルタワーホール

セッションを聞いていた考えたことを残しておく。

# kubefork - 自分専用のクラスタを所有しているような開発体験

https://event.cloudnativedays.jp/cndt2022/talks/1590

cndt2020 で「[TelepresenceとIstioを使った自分専用のクラスタを持っているような開発環境の構築](https://event.cloudnativedays.jp/cndt2020/talks/17)」を見た記憶があり、その続きのようなセッションだった。

[kubefork](https://github.com/wantedly/kubefork-controller) という特定のリクエストのみルーティングする仕組みについての解説で、聞いている途中に context 伝搬は sidecar 的な仕組みでは難しいのではないかと思っていたが、改善案として Header Propagation 機能の提供が話されていたため、やはりシンプルには解決できなさそうだった。

以前「[Dapr の概念と実装から学ぶ Observability への招待](https://event.cloudnativedays.jp/o11y2022/talks/1382)」を聞いた時に dapr の tracing に関しても context 伝搬について考えた記憶がある。[Trace Context](https://www.w3.org/TR/trace-context/) が標準化されていて vendor 固有の情報を入れられるという話だったと思うので、そこで色々な context 伝搬すると応用が効きそうに思ったけど tracing context がそのようなものであるのかは分かってない。

このセッションは前回の続き物という感じであり、過去別分野のセッションでも似たようなことを考えた記憶があり、面白いと感じられたセッションだった。

# DMMプラットフォームを支える負荷試験基盤

https://event.cloudnativedays.jp/cndt2022/talks/1511

今回 DMM のセッションが多く見かけたが、その中で個人的には負荷試験についてが面白かった。

大雑把には [locust](https://github.com/locustio/locust) を用いた共通基盤についての話だったが、課題としてアクセスログ出力による費用増が挙げられているのが特に興味深かった。
個人的に経験した負荷試験では test double 的に外部への依存は置き換えがされていた。そういった場合にはこの問題はそこまで大きくならないと思う。
一方で、DMM で行われていたのはリクエストのヘッダーでアクセスログの出力を制御することだった。ただし、この場合には依存先のアプリケーションのログ抑制は難しいとされていた。この時に考えたのもやはり context 伝搬であり、上のセッションで考えたことが再び頭をよぎった。

この話を聞くと他のチームがどのようにして負荷試験を行っているのか気になり、どうすれば良い負荷試験が出来そうか考えるのが面白く感じられるセッションだった。

# 実践！OpenTelemetry と OSS を使った Observability 基盤の構築

https://event.cloudnativedays.jp/cndt2022/talks/1571

Grafana 系で 3 pillars + correlation の実装詳細が面白かった。

Line Developer Day 2021 の「[Kubernetesにおける可観測性プラクティスの実装](https://linedevday.linecorp.com/2021/ja/sessions/25/)」でも似たような構成で実現した話を聞いた記憶があったが、今回のセッションを聞くことでそれがより補完された感じがある。

一つ気になったのは `SpanMetrics` というものだった。

https://grafana.com/docs/tempo/latest/grafana-agent/span-metrics/

> Span metrics allow you to generate metrics from your tracing data automatically. Span metrics aggregates request, error and duration (RED) metrics from span data. Metrics are exported in Prometheus format.

初めて 3 pillars を知った「[Observability 3 ways: Logging, Metrics & Tracing](https://www.youtube.com/watch?v=juP9VApKy_I)」がそれぞれの特性の理解の元となっているが、metrics の良いところは本来は膨大なイベントから限られた情報量で傾向を表現できるところだと思っており、それを span のようなサンプリングと切り離せなさそうなイベントを元にしてしまうと良いところが薄れるのではというのが第一印象としてあった。

ただし、それは実装次第でもありそうだし、

> Span metrics are of particular interest if your system is not monitored with metrics, but it has distributed tracing implemented. You get out-of-the-box metrics from your tracing pipeline.

> Even if you already have metrics, span metrics can provide in-depth monitoring of your system. The generated metrics show application-level insight into your monitoring, as far as tracing gets propagated through your applications.

とも書かれているのでより補強する便利なものとして考えるで良いのかもしれない。

このように `SpanMetrics` が個人的には新鮮に感じられたので面白かった。

# 終わり

今年の CloudNative Days Tokyo は久しぶりにオフライン参加をしたけどオンラインで参加している人も多そうだったし、完全に以前のように戻ったという感じではなかった。それでも楽しいセッションがあり興味も尽きないので、来年も楽しみにしている。 
