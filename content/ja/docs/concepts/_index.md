---
title: コンセプト
main_menu: true
content_template: templates/concept
weight: 40
---

{{% capture overview %}}

このコンセプトのセクションはKubernetesシステムの一部と、Kubernetesがクラスタを表現するために使う抽象概念に
ついて学び、Kubernetesがどのように動作するのかについてのより深い理解を得る手助けします。

{{% /capture %}}

{{% capture body %}}

## 概要 {#overview}

Kubernetesを使うには、クラスタの *求める状態* を記述するために *Kubernetes APIオブジェクト* を使います。
実行したいアプリケーションやその他のワークロード、それらが使うコンテナイメージ、レプリカ数、利用したい
ネットワークやディスクリソースなどを記述します。通常は`kubectl`コマンドラインインタフェース経由で、
Kubernetes APIを使ってオブジェクトを作成することで求める状態を設定します。Kubernetes APIを直接使ってクラスタと
やりとりし、求める状態を設定したり修正したりもできます。

求める状態を設定すると、クラスタの現在の状態を求める状態にマッチさせるために *Kubernetesコントロールプレーン*
が動作します。そのために、Kubernetesはさまざまなタスクを自動で実行します。コンテナの起動や再起動、
アプリケーションのレプリカ数のスケーリングなどです。Kubernetesコントロールプレーンはクラスタで実行するプロセスの
集合で構成されます。

* **Kubernetesマスタ** はクラスタの単一ノードで実行する3つのプロセスの集合で、マスタノードと呼ばれます。
  これらのプロセスは[kube-apiserver](/docs/admin/kube-apiserver/)、
  [kube-controller-manager](/docs/admin/kube-controller-manager/)、[kube-scheduler](/docs/admin/kube-scheduler/)
  です。
* マスタでないノードではそれぞれ、以下の2つのプロセスを実行します
  * **[kubelet](/docs/admin/kubelet/)**: Kubernetesマスタと通信します
  * **[kube-proxy](/docs/admin/kube-proxy/)**: 各ノードでのKubernetesネットワーキングサービスを反映する
    ネットワークプロキシです

## Kubernetesオブジェクト {#kubernetes-objects}

Kubernetesにはシステムの状態を表現する数多くの抽象概念が含まれます。デプロイされたコンテナ化アプリケーションと
ワークロード、それらに関連するネットワークとディスクリソース、クラスタが行ったことについての情報などがあります。
これらの抽象概念はKubernetes APIのオブジェクトで表現されます。詳細は
[Kubernetesオブジェクトの概要](/docs/concepts/abstractions/overview/)を参照してください。

基本的なKubernetesオブジェクトには以下があります。

* [Pod](/ja/docs/concepts/workloads/pods/pod-overview/)
* [Service](/docs/concepts/services-networking/service/)
* [Volume](/ja/docs/concepts/storage/volumes/)
* [名前空間](/ja/docs/concepts/overview/working-with-objects/namespaces/)

加えて、コントローラと呼ばれる高水準の抽象概念も数多くあります。コントローラは基本オブジェクトの上に構築され、
追加の機能や便利な機能を提供します。

* [ReplicaSet](/docs/concepts/workloads/controllers/replicaset/)
* [Deployment](/docs/concepts/workloads/controllers/deployment/)
* [StatefulSet](/ja/docs/concepts/workloads/controllers/statefulset/)
* [DaemonSet](/ja/docs/concepts/workloads/controllers/daemonset/)
* [Job](/ja/docs/concepts/workloads/controllers/jobs-run-to-completion/)

## Kubernetesコントロールプレーン {#kubernetes-control-place}

Kubernetesマスタやkubeletプロセスといった、Kubernetesコントロールプレーンはクラスタとどのように通信するのかを
管理します。コントロールプレーンはシステムのすべてのKubernetesオブジェクトの記録を維持し、これらのオブジェクトの
状態を管理する継続的な制御ループを実行します。コントロールプレーンの制御ループは、いつでもクラスタでの変更に
反応し、すべてのオブジェクトの現在の状態を求める状態にマッチさせるために動作します。

例えば、Deploymentオブジェクトを生成するためにKubernets APIを使う場合、システムに新たな求める状態を提供します。
Kubernetesコントロールプレーンはオブジェクトの作成を記録し、必要なアプリケーションを起動し、それらをクラスタノードに
スケジューリングすることで指示を遂行します。このように、クラスタの実際の状態を求める状態にマッチさせます。

### Kubernetes Master {#kubernetes-master}

Kubernetesマスタはクラスタに対して、求める状態を維持する責任を負っています。`kubectl`コマンドライン
インタフェースを使うなどしてKubernetesとやりとりする場合、それはクラスタのKubernetesマスタと通信することに
なります。

> 「マスタ」はクラスタの状態を管理しているプロセスの集合のことです。通常、これらのプロセスはすべてクラスタの
> 単一のノードで実行されており、このノードがマスタと呼ばれます。マスタは可用性と冗長性のために複製することも
> できます。

### Kubernetes Nodes {#kubernetes-nodes}

クラスタのノードはアプリケーションやクラウドワークフローを実行するマシン (VM、物理サーバなど) です。
Kubernetesマスタは各ノードを制御します。ユーザがノードと直接やりとりすることはめったにありません。

#### オブジェクトメタデータ {#object-metadata}

* [Annotations](/docs/concepts/overview/working-with-objects/annotations/)

{{% /capture %}}

{{% capture whatsnext %}}

If you would like to write a concept page, see
[Using Page Templates](/docs/home/contribute/page-templates/)
for information about the concept page type and the concept template.

{{% /capture %}}
