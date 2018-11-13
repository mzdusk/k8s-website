---
layout: docsportal
css: /css/style_user_journeys.css
js: https://use.fontawesome.com/4bcc658a89.js, https://cdnjs.cloudflare.com/ajax/libs/prefixfree/1.0.7/prefixfree.min.js
title: Foundational
track: "USERS › CLUSTER OPERATOR › FOUNDATIONAL"
content_template: templates/user-journey-content
---

{{% capture overview %}}

Kubernetesクラスタの管理と運用を始める方法を学びたいのであれば、このページとリンク先のトピックで基礎的な概念と
タスクを紹介します。このページではKubernetesクラスタと主要な概念を紹介します。内容はクラスタ自身よりも、まずは
そこで動くソフトウェアにフォーカスします。

{{% /capture %}}

{{% capture body %}}

## Kubernetesの概要

まだであれば、[Kubernetesとは](/ja/docs/concepts/overview/what-is-kubernetes/)を読んで理解を深めてください。
ここでは数多くの基本的な概念や用語を紹介します。

Kubernetsは非常に柔軟なので、クラスタは様々な場所で実行できます。自身のラップトップや仮想マシンの中で動作する
ローカルの開発機からKubernetesとやりとりができます。Kubernetesはローカルやクラウドプロバイダでホストされている
仮想マシン上でも実行できますし、ベアメタル上でも実行できます。

クラスタは1つもしくは複数の[ノード](/ja/docs/concepts/architecture/nodes/)で構成されます。ノードは物理もしくは
仮想マシンです。複数のノードがあれば、それらのノードは
[クラスタネットワーク](/docs/concepts/cluster-administration/networking/)で接続されます。ノードの数にかかわらず、
すべてのKubernetesクラスタは通常同じコンポーネントを持ち、これらは
[Kubernetesコンポーネント](/docs/concepts/overview/components)で詳述します。


## Kubernetesの基本について学ぶ

Kubernetesクラスタの管理と運用に詳しくなる良い方法は、セットアップすることです。クラスタを試す最もコンパクトな
方法の1つは[Minikubeをインストールして使うこと](/docs/tasks/tools/install-minikube/)です。Minikubeはローカルの
ラップトップまたは開発機上の仮想マシンで単一ノードクラスタをセットアップし実行するためのコマンドラインツールです。
Minikubeは[Katacoda Kubernetes Playground](https://www.katacoda.com/courses/kubernetes/playground)でブラウザ上
でも利用できます。Katacodaは単一ノードクラスタへのブラウザベースの接続を提供しており、裏でminikubeを使い、
Kubernetesを探索するための多くのチュートリアルをサポートしています。同じ目的でウェブベースの
[Play with Kubernetes](http://labs.play-with-k8s.com/)を活用することもできます。

Kubernetesとはダッシュボード、API、Kubernetes APIとやりとりする (`kubectl`のような) コマンドラインツールを
使ってやりとりできます。構成ファイルを使って
[クラスタアクセスを構造化](/docs/concepts/configuration/organize-cluster-access-kubeconfig/)することに慣れて
ください。Kubernetes APIは、Kubernetes上でソフトウェアを動作させるために使う構成要素や抽象概念を提供する多くの
リソースを公開しています。これらのリソースについての詳細は
[Kubernetesオブジェクトの理解](/docs/concepts/overview/working-with-objects/kubernetes-objects)参照してください。
これらのリソースはKubernetesドキュメントにある数多くの記事で取り上げられています。

* [Pod Overview](/ja/docs/concepts/workloads/pods/pod-overview/)
  * [Pods](/docs/concepts/workloads/pods/pod/)
  * [ReplicaSets](/docs/concepts/workloads/controllers/replicaset/)
  * [Deployments](/docs/concepts/workloads/controllers/deployment/)
  * [Garbage Collection](/docs/concepts/workloads/controllers/garbage-collection/)
  * [Container Images](/docs/concepts/containers/images/)
  * [Container Environment Variables](/docs/concepts/containers/container-environment-variables/)
* [Labels and Selectors](/docs/concepts/overview/working-with-objects/labels/)
* [名前空間](/ja/docs/concepts/overview/working-with-objects/namespaces/)
  * [Namespaces Walkthrough](/docs/tasks/administer-cluster/namespaces-walkthrough/)
* [Services](/docs/concepts/services-networking/service/)
* [Annotations](/docs/concepts/overview/working-with-objects/annotations/)
* [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/)
* [Secrets](/docs/concepts/configuration/secret/)

クラスタ管理者として、これらすべてのリソースを使う必要はないかもしれませんが、どのようにクラスタを使うのかを
理解するために、これらについて詳しくなっておくべきです。留意すべき追加のリソースが数多くあり、いくつかは
[中級のリソース](/ja/docs/user-journeys/users/cluster-operator/intermediate#section-1)に列挙しています。
[Kubernetesリソースの管理方法](/docs/concepts/cluster-administration/manage-deployment/)にも詳しくなっておくべきです。

## クラスタ情報の取得

[Kubernetes APIを使ってクラスタにアクセス](/docs/tasks/administer-cluster/access-cluster-api/)できます
。まだこの方法に詳しくないのであれば、[入門チュートリアル](/docs/tutorials/kubernetes-basics/explore-intro/)を
見直してください。`kubectl`を使うことで、非常に素早くKubernetesクラスタの情報を得ることができます。クラスタ内の
ノードについての基本的な情報を得るには、`kubectl get nodes`を実行します。同じノードについてのより詳細な情報は、
`kubectl describe nodes`で得られます。Kubernetesのコアの状態は`kubectl get componentstatuses`で見られます。

クラスタについての情報を得るためのリソースとその操作方法は以下を参照してください。

* [計算資源、ストレージ資源、ネットワーク資源をモニタリングするツール](/docs/tasks/debug-application-cluster/resource-usage-monitoring/)
* [コアメトリクスパイプライン](/docs/tasks/debug-application-cluster/core-metrics-pipeline/)
  * [メトリクス](/docs/concepts/cluster-administration/controller-metrics/)

## 追加リソースの探索

### チュートリアル

* [Kubernetesの基本](/ja/docs/tutorials/kubernetes-basics/)
* [Kubernetes 101](/docs/user-guide/walkthrough/) - kubectlコマンドラインインタフェースとPod
* [Kubernetes 201](/docs/user-guide/walkthrough/k8s201/) - ラベル、Deployment、サービス、ヘルスチェック
* [ConfigMapによるRedisの構成](/docs/tutorials/configuration/configure-redis-using-configmap/)
* ステートレスアプリケーション
  * [Redisを使ったPHP Guestbookのデプロイ](/ja/docs/tutorials/stateless-application/guestbook/)
  * [アプリケーションへアクセスするための外部IPの公開](/docs/tutorials/stateless-application/expose-external-ip-address/)

{{% /capture %}}
