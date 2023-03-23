---
title: "CI/CD Conference 2023 recap"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cicd2023"]
published: true
---

CI/CD Conference 2023 にオフライン参加したので　recap。

# 概要

日程: 2023/03/20
会場: 日比谷国際ビル コンファレンススクエア

霞ヶ関は桜が綺麗だった。
参加者は 1300 人くらいで、参加比率はオンライン:オフラインで 4.75:1 くらいだったらしい。 
[前回](cloudnativedaystokyo2022.md) は　5:1 くらいで若干増えたらしい。
今後のハイブリットイベントはこれくらいの比率がデフォルトになるんだろうか。

# レポート
## アプリケーション開発者の体験

アプリケーション開発者の体験についていくつかのセッションで言及されていた。
全体的なメッセージとしてインフラの人ではなくてアプリケーションの人にもっと参加していってもらいたいというものがあり、傾向を汲んでのものだったのかもしれない。

- 時間
- 認知負荷

が共通して挙げられることが多くあった。

時間についてはテストでのコンテナのチューニングやキャッシュなどの方法が各セッションで紹介されている。

認知負荷については CDK のセッションで語られた内容が興味深く、CDK (TypeScript) を採用した理由として、開発者が AWS SDK + TypeScript に慣れているため同じ感覚で扱えるというものがあった。
imperative, declarative, programmable と種類がある ui の内で programmable なツールが登場した背景として開発者の学習コストの削減というものをよく聞くため単純に認知負荷が下がるというのは頭では理解していたが、特に CDK のようなものだと AWS SDK と同じ感覚で扱えるというのは面白く感じた。

## セキュリティ

セキュリティについては特にスポンサーセッションで多く取り上げられていた。

Snyk, SonarQube, NeuVector などいろいろなソフトウェアがあり、各種インテグレーションや組み合わせて使ったりというデモがなされた。
色々と組み合わせることでパイプライン一通りをカバーすることは現状できるように思えたが、個人的には Observability であったことと同じく [^1] セキュリティ情報が各ソフトウェアに分断されるのではなく一つで見ることが出来た方が嬉しいと感じている。
ただし、そこまでできるのかは分かっていない。

[^1]: 情報として相関などないなら Observability よりは需要がないのかもしれない

# セッションについて
## 大規模レガシーテストを倒すためのCI基盤の作り方

https://event.cloudnativedays.jp/cicd2023/talks/1773

- Cloud Build + GitHub (ghec) + Perl + Go での CI 基盤
    - GitHub (ghec) との連携（特に Go のプライベートモジュール）は Cloud Functions 経由 で GitHub App でトークンを取得
    - Cloud Build では `skip ci` が含まれるコミットメッセージが特別処理される
        - GitHub Webhook → Functions → Pub/Sub → Build というトリガーで回避
- 大規模テストの課題
    - テストはしっかり書かれているが古い＆多い → テスト時間
        - *単純にトリガーを分けるのは checks が溢れて開発者体験が悪い*
        - 一つのトリガーからテストジョブを並列で起動 → 並列しすぎて全体としての時間が長い
            - LA などのリソースの metrics を見て決定: LA1 が 5 を超えない (8 core)
            - CPU の割り当て最適化
                - 対象: perl + MySQL 等の sidecar
                - `docker run --cpuset-cpus 5-7` コンテナが使う CPU を固定する (pinning)
                - `docker run --cpu-shares $((1024*4))` コンテナの CPU 割り当ての weight を上げる (default 1024)
                - 結果: このケースでは MySQL に優先して割当たるようにすると改善
            - ホストネットワークの利用: `net=host` と `net=cloudbuild` で若干差が出る
- 学び: ボトルネック → 計測 → チューニング

## 最高の開発者体験を目指してAWS CDKでCI/CDパイプラインを改善し続けている話

https://event.cloudnativedays.jp/cicd2023/talks/1777

- 前半
    - ユーザーに価値を届けるためにも効率化・自動化は重要 → 最高の開発者体験を追求
    - インフラ構成とデプロイに伴う課題
        - EC2 中心のサービス → コンテナ化がほぼ完了のフェーズ
        - EC2 時代: 安全かつ高速とは言い難かった
            - リリース前にサーバーへ一斉に展開、シェルスクリプトでローリングアップデート
            - デプロイする段階でビルドが実行、環境間でアセットを共有していないため再度ビルド
        - CDK (TypeScript) を採用: *開発者が AWS SDK + TypeScript に慣れているため同じ感覚で扱える*
        - ECS 時代
            - Canary: 通常のユーザーは 1/20 だけアクセス、ヘッダーで開発者はアクセス制御
            - デプロイメントサーキットブレーカー: https://dev.classmethod.jp/articles/awssummit2021-ecs-deployment-circuit-breaker/
                - > ECSサービスのデプロイで異常が発生した場合に、以前にデプロイが成功していたバージョンに自動でロールバックする機能
            - ChatOps: Slack → Deploy (canary) → e2e → Slack → Approve → Apply
        - *パイプラインの改善に開発者が積極的に参加するにはまだ仕組みの認知負荷が高い*
- 後半
    - 開発者全員が SRE (25 名くらいの規模) を実現する
    - CDK + TypeScript がおすすめ
        - バックエンドを Go で書いていたので Go で初めたが CDK in Go は辛い
            - ポインタへの変換
            - err を返さず panic する
            - エスケープハッチに実質非対応: https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/cfn_layer.html

## UbieはなぜSnykを選んだのか？安全で高速なアプリケーション開発ライフサイクルの実現へ

https://event.cloudnativedays.jp/cicd2023/talks/1795

- 前半: snyk スポンサー
    - https://docs.snyk.io/integrations
        - vscode demo
- 後半: ubie
    - 課題
        - 組織全体の脆弱性管理
        - すぐに修正できない場合 (3rd party package) の状態管理
        - その他: フルマネージド / 脆弱性修正のサポート / リスクの高い脆弱性 / API 連携
    - 過去: 内製ツールの運用 (https://github.com/m-mizutani/octovy) → 工数避けず断念
    - snyk
        - Open Fix PR
        - 各種インテグレーション / Slack / JIRA
        - 運用: 脆弱性対応ガイドライン / リスク評価の推奨頻度

## インフラCI/CDの継続的改善の道のり

https://event.cloudnativedays.jp/cicd2023/talks/1755

- COLOR ME: アプリエンジニア 30 / SRE 6
    - オンプレ → OpenStack → k8s on OpenStack → hybrid
- 技術スタック
     - VM
        - OpenStack → terraform
        - Middle → Puppet
        - Local (vm apply) → コンテナ puppet apply + serverspec → 各種環境
    - Kubernetes
        - GitOps: main branch → staging → release branch → production
- 課題
    - 時間が掛かる (x10 min)
    - メンテナンスが大変 （時間と共にシステムは壊れていく）
    - コードとサーバーの差分 （長期間デプロイされていない / アドホックな対応 → 自動化への恐怖）
    - 暗黙知前提の運用 （手順書のメンテナンス / 各自のローカルから実行 / コード変更の作業 = SRE への依頼）
    - VM コンテナ混在 （VM 向けに最適化された設定）
    - セキュリティ対応
    - 学習コスト （さまざまな言語 / さまざまなツール）
- アプローチ: *アプリケーション開発者ができる*
    - 高速化
        - GitHub Actions のチューニング （cache / 並列化） (2-3 min)
        - パイプライン自体の最適化 / kernel ヘッダーや gcc のアップデートなど本筋ではない部分をキャッシュ (10-15 min)
    - 暗黙知
        - ドキュメンテーション、勉強会、ワークショップ
        - CI に関する作業の自動化
            - ローカルからの排除
            - オプションの最小化 (Makefile)
            - 人がラベルを設定するのではなく修正したコードファイルを見て自動化
            - 構成図: https://github.com/k1LoW/ndiag
    - 構成ドリフト検出
        - 定期的に差分検出 + コード適用
        - 複数のデプロイフローを統一し変更履歴を管理
    - セキュリティ
        - Dockerfile latest 指定 → daily staging で動作確認 → production
        - yum cron で部分適用 → 全体適用
        - 拾いきれない脆弱性は専属メンバーをアサイン / 1w 交代
    - 権限移譲: SRE チームの作業負荷が下がる = アプリケーションエンジニアの負荷が上がるにならないように
- 今後
    - four keys / immutable infra / 継続的にアップデート

## OSSでセキュリティをCI/CDパイプラインに透過的に取込む方法

https://event.cloudnativedays.jp/cicd2023/talks/1796

- suse
    - cicd / 静的解析 / イメージの脆弱性スキャン / イメージのコンプライアンススキャン / アドミッション制御 / Security Policy as Code
    - demo
        - commit → GitHub → sonarqube (static analysis) → harbor → neuvector (image scan) → helm → kubernetes → neuvector (runtime)

## "State of DevOps" ウェブアプリケーションのdeliveryを考えるとき、今何をすればいいのか(実践編)

https://event.cloudnativedays.jp/cicd2023/talks/1771

- State of Devops / Lean と DevOps の科学
    - CircleCI が日本語訳してくれている
    - fourkeys 2022: elite が消えた
    - cap 24: トランクベースの開発手法の実践 (2022 で別の傾向が出ている)
- 実践
    - コンテナ化 / IaC / build 自動化 / unit integration e2e テストの自動化 / セキュリティのシフトレフト / デプロイの人力を減らす / 開発者の心理的負荷を下げる（ユーザー影響の低減） / 個人環境の独立
- freee の事例 (手動 6 step)
    - Slack → PR 作成 → Merge → build → manifest PR & auto Merge → ArgoCD → staging → comitter manual testing (slack notification) + auto e2e
    - Slack → PR 作成 → Merge → manifest PR → merge → ArgoCD → production → canary (5% / 30 min)
- ロードマップ
    - four keys
        - デプロイ回数を増やす: 完全自動化 & 1 PR 1 deploy
        - MTTR の改善: 自動ロードバック & observability 向上 + circuit breaker の実装
    - CICD 以外の cap の改善
        - 従業員ロイヤリティの向上
        - リーダーシップ
    - devops team → platform team

# 後で見る
## トランクベース開発の実現に向けた開発プロセスとCIパイプラインの継続的改善

https://event.cloudnativedays.jp/cicd2023/talks/1774

トランクベース開発について知りたい & `"State of DevOps" ウェブアプリケーションのdeliveryを考えるとき、今何をすればいいのか(実践編)` で 2022 年ではトランクベース開発の傾向に変化があったと聞いたため。

## Kubernetesリソースの安定稼働に向けた　TerratestによるHelmチャートのテスト自動化

https://event.cloudnativedays.jp/cicd2023/talks/1767

単純に Terratest が面白そうだったため。

## DORA 2022 度版 State of DevOps Report

https://zenn.dev/google_cloud_jp/articles/83f6667dfc9519

2022 年の State of DevOps Report について探していたら分かりやすい記事を見つけた。
