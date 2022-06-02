---
title: "GitHub Codespaces と Gitpod を使う"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "gitpod", "codespaces"]
published: true
---

[GitHub InFocus 2022](https://resources.github.com/webcast/jp-github-infocus-2022) で Codespaces について話す機会があったけど話しきれなかったのと、 GitHub Codespaces が個人で使えるようになったのでそれぞれ使ってみて違いを感じた感想を書く。

# まとめ

| Features | Codespaces | Codespaces (Enterprise) | Gitpod |
| --- | --- | --- | --- |
| Kubernetes | o | o | x |
| Prebuild | x | o | o |
| Copilot | o | o | x |

o は実際に試して使えた、x は試したけどやり方が分からなかった機能。

Codespaces は個人で使った環境、Codespaces (Enterprise) は Enterprise で使った環境、Gitpod は個人で使った環境を表す。

Kubernetes は環境内で Minikube などが立ち上げられるかどうか。

Prebuild はコンテナイメージなどを事前にビルドして、すぐに環境を立ち上げることができるかどうか。

Copilot は GitHub Copilot が有効化できるかどうか。

# Kubernetes

Codespaces では例えば [Dev container features (preview)](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview) で minikube を選択すれば使える。

Gitpod でのやり方は分からなかった。
[Root, Docker and VS Code](https://www.gitpod.io/blog/root-docker-and-vscode) という投稿はあったが、Kind を立ち上げようとしても上手くいかなかったので実現できていない。

# Prebuild

- Codespaces: [Prebuilding your codespaces](https://docs.github.com/en/codespaces/prebuilding-your-codespaces)
- Gitpod: [Prebuilds](https://www.gitpod.io/docs/prebuilds)

Prebuild 自体はどちらもあるが、個人の Codespaces だとリポジトリに設定が出てこなかったので使えなかった。

# Copilot

Gitpod の Extensions では [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) が検索しても出て来ない。

# Others

実際に試していないけど違いを感じる部分としては、プライベートなネットワークへの接続がある。

例えば GitHub Actions ではそういった場合は [Self-hosted runners](https://docs.github.com/ja/actions/hosting-your-own-runners/about-self-hosted-runners) で解決しているが、Codespaces には self-hosted runners のような機能はない。

一方で、Gitpod では [Self-hosted](https://www.gitpod.io/self-hosted) がある。

Codespaces の場合は [Connecting to a private network](https://docs.github.com/en/codespaces/developing-in-codespaces/connecting-to-a-private-network) として VPN を利用した方法が紹介されていたけど試してはいない。

実際この問題については、ローカルの vscode remote container で開発をすることで回避していた。
