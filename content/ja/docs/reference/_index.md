---
title: リファレンス
linkTitle: "リファレンス"
main_menu: true
weight: 70
content_template: templates/concept
---

{{% capture overview %}}

このセクションのKubernetesドキュメントにはリファレンスが含まれます。

{{% /capture %}}

{{% capture body %}}

## APIリファレンス {#api-reference}

* [Kubernetes API Overview](/docs/reference/using-api/api-overview/) - KubernetesのAPIの概要
* Kubernetes APIバージョン
  * [1.12](/docs/reference/generated/kubernetes-api/v1.12/)
  * [1.11](/docs/reference/generated/kubernetes-api/v1.11/)
  * [1.10](https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/)
  * [1.9](https://v1-9.docs.kubernetes.io/docs/api-reference/v1.9/)
  * [1.8](https://v1-8.docs.kubernetes.io/docs/api-reference/v1.8/)
  * [1.7](https://v1-7.docs.kubernetes.io/docs/api-reference/v1.7/)

## APIクライアントライブラリ {#api-client-library}

プログラミング言語からKubernetes APIを呼び出すために、[クライアントライブラリ](/docs/reference/using-api/client-libraries/)が使えます。公式にサポートしているクライアントライブラリは次の通りです。

- [Kubernetes Goクライアントライブラリ](https://github.com/kubernetes/client-go/)
- [Kubernetes Pythonクライアントライブラリ](https://github.com/kubernetes-client/python)

## CLIリファレンス {#cli-reference}

* [kubectl](/docs/user-guide/kubectl-overview) - コマンドの実行とKubernetesクラスタの管理を行う、メインのCLIツール
    * [JSONPath](/docs/user-guide/jsonpath/) - kubectlで[JSONPath表現](http://goessner.net/articles/JsonPath/)を使う際の文法ガイド
* [kubeadm](/docs/admin/kubeadm/) - セキュアなKubernetesクラスタを容易にプロビジョニングするためのCLIツール
* [kubefed](/docs/admin/kubefed/) - 連合クラスタの管理を手助けするCLIツール

## 構成リファレンス {#config-reference}

* [kubelet](/docs/admin/kubelet/) - 各ノードで実行する最も重要な *ノードエージェント* です。kubeletはPodSpecのセットを受け取り、記述されたコンテナが正常に実行されることを保証します。
* [kube-apiserver](/docs/admin/kube-apiserver/) - PodやService、ReplicationコントローラなどのAPIオブジェクトに対するデータの検証と構成を行うREST APIです。
* [kube-controller-manager](/docs/admin/kube-controller-manager/) - Kubernetesに同梱される、コア制御ループが組み込まれたデーモンです。
* [kube-proxy](/docs/admin/kube-proxy/) - バックエンドに対して、単純なTCP/UDPストリーム転送やラウンドロビンTCP/UDP転送を行います。
* [kube-scheduler](/docs/admin/kube-scheduler/) - 可用性、パフォーマンス、キャパシティを管理するスケジューラです。
* [federation-apiserver](/docs/admin/federation-apiserver/) - 連合クラスタのAPIサーバです。
* [federation-controller-manager](/docs/admin/federation-controller-manager/) - Kubernetes federationに同梱されるコア制御ループが組み込まれたデーモンです。

## 設計ドキュメント {#design-docs}

Kubernetesの機能に対する設計ドキュメントのアーカイブです。[Kubernetes Architecture](https://git.k8s.io/community/contributors/design-proposals/architecture/architecture.md)と[Kubernetes Design Overview](https://git.k8s.io/community/contributors/design-proposals)から始めるのが良いでしょう。

{{% /capture %}}
