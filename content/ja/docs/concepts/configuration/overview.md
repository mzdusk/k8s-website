---
title: 構成ベストプラクティス
content_template: templates/concept
weight: 10
---

{{% capture overview %}}
この文書はユーザガイドやGetting Started、例の至る所で紹介されている構成ベストプラクティスをハイライトし統合します。

これは生きた文書です。このリストにないが他で便利な何かを思い付いたのであれば、IssueやPRを提出するのをためらわないでください。
{{% /capture %}}

{{% capture body %}}
## 一般的な構成のTips

- 構成を定義するときは、最新の安定したAPIバージョンを指定する。

- 構成ファイルはクラスタにpushする前にバージョンコントロールに格納すべきである。これにより必要があれば構成の変更を素早くロールバックできる。クラスタの再作成や復旧の助けにもなる。

- 構成ファイルはJSONではなくYAMLで書く。これらのフォーマットはほとんどの場合同じ意味で使えるが、YAMLの方がユーザフレンドリである。

- 関連するオブジェクトは筋が通るのであれば単一のファイルにまとめる。単一ファイルは分割しているよりも大抵管理しやすい。この文法の例は [guestbook-all-in-one.yaml](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/guestbook/all-in-one/guestbook-all-in-one.yaml) のファイルを参照。

- 多くの`kubectl`コマンドはディレクトリで呼び出せることにも注意。例えば、構成ファイルのディレクトリで`kubectl create`を呼ぶことができる。

- 必要のないデフォルト値を指定しない。シンプルで最小の構成がミスを少なくできる。

- わかりやすいように、アノテーションにオブジェクトの説明を書く


## "裸の"Pod 対 ReplicaSet, Deployment, Job

- 避けることができるのなら、裸のPod (すなわち、[ReplicaSet](/ja/docs/concepts/workloads/controllers/replicaset/)や[Deployment](/docs/concepts/workloads/controllers/deployment/)に関連していないPod) を使わないこと。裸のPodはノード障害時に再スケジュールされない。

  常に利用可能なPodの数を保証するためのReplicaSetを作成し、([RollingUpdate](/ja/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)のような)Podを置き換える戦略を指定するDeploymentは、明示的な[`restartPolicy: Never`](/ja/docs/concepts/workloads/pods/pod-lifecycle/#再起動ポリシ)シナリオを除く、ほぼすべての場合でPodを直接作成するよりも望ましい。[Job](/ja/docs/concepts/workloads/controllers/jobs-run-to-completion/)が適切な場合もある。


## サービス

- サービスに付随するバックエンドのワークロード (DeploymentやReplicaSet) の前と、サービスにアクセスする必要のあるワークロードの前に、その[サービス](/ja/docs/concepts/services-networking/service/)を作成する。Kubernetesがコンテナを開始すると、コンテナが開始された時に起動しているすべてのサービスを指す環境変数を提供する。たとえば、サービス名 `foo` が存在していると、すべてのコンテナは初期環境で以下の変数を得られる。

  ```shell
  FOO_SERVICE_HOST=<the host the Service is running on>
  FOO_SERVICE_PORT=<the port the Service is running on>
  ```

  もしサービスとやりとりするコードを書いているのであれば、これらの環境変数は使わず、かわりに[サービスのDNS名](/ja/docs/concepts/services-networking/dns-pod-service/)を使うこと。サービス環境変数はDNS検索を修正できない古いソフトに対してのみ提供されており、はるかに柔軟性に乏しいサービスへのアクセス方法である。
  
- 本当に必要でないのなら、Podに対して`hostPort`を指定しないこと。Podが`hostPort`に結合されると、各<`hostIP`, `hostPort`, `protocol`>の組み合わせは一意でなければならないので、 Podをスケジュールできる場所の数が制限される。`hostIP`と`protocol`を明示的に指定しなければ、Kubernetesはデフォルトの`hostIP`に`0.0.0.0`を、デフォルトの`protocol`に`TCP`を使う。

  デバッグ目的でポートにアクセスする必要があるときのみ、[apiserverプロキシ](/ja/docs/tasks/access-application-cluster/access-cluster/#manually-constructing-apiserver-proxy-urls)または[`kubectl port-forwarding`](/ja/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)を使う
  
  明示的にPodのポートを公開する必要があるときのみ、`hostPort`に頼る前に[NodePort](/ja/docs/concepts/services-networking/service/#nodeport)サービスの使用を検討すること。

- `hostPort`と同じ理由で、`hostNetwork`の使用は避けること。

- `kube-proxy`ロードバランシングが必要なければ、サービスディスカバリを簡単にするために[ヘッドレスサービス](/ja/docs/concepts/services-networking/service/#headless-services) (`ClusterIP`が`None`であるもの)を使うこと。


## ラベルの使用

- `{ app: myapp, tier: frontent, phase: test, deployment: v3 }` のように、アプリケーションもしくはDeploymentの __意味的な属性__ を識別する[ラベル](/ja/docs/concepts/overview/working-with-objects/labels/)を定義し、使うこと。これらのラベルは、他のリソースから適切なPodを選択するために使われる。例えば、すべての `tier: frontent` Pod や、`app: myapp`のすべての`pahse: test`コンポーネントを選択するサービスである。このアプローチの例は[guestbook](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/guestbook/)アプリを参照。

サービスはセレクタからリリース指定ラベルを省略することで、複数のDeploymentを分割することができる。[Deployment](/ja/docs/concepts/workloads/controllers/deployment/)はダウンタイムなしで実行中のサービスの更新を容易にする。

オブジェクトの望ましい状態はDeploymentに記述され、スペックへの変更が _適用_ されると、Deploymentコントローラは制御された割合で現在の状態から望ましい状態へ変更する。

- デバッグのためにラベルを操作できる。(ReplicaSetのような) Kubernetesコントローラとサービスはセレクタラベルを用いてPodにマッチするので、Podから関連するラベルを削除することでコントローラによる監視と、サービスからのトラフィックを止められる。既存のPodからラベルを削除すると、コントローラはそれを置き換える新しいPodを作成する。これは"隔離された"環境で以前は"生きていた"Podをデバッグするのに便利な方法である。インタラクティブにラベルを追加または削除するには、[`kubectl label`](/docs/reference/generated/kubectl/kubectl-commands#label)を使う。


## コンテナイメージ {#container-images}

[imagePullPolicy](/docs/concepts/containers/images/#updating-images)とイメージのタグは、[kubelet](/docs/admin/kubelet/)が特定のイメージをPullしようとするときに影響を与えます。

- `imagePullPolicy: IfNotPresent`: ローカルにイメージが存在しない場合にのみPullします

- `imagePullPolicy: Always`: Podが起動すると、毎回イメージをPullします

- `imagePullPolicy` がなく、イメージのタグが`:latest`か指定がない: `Always`が適用されます。

- `imagePullPolicy` がなく、イメージのタグに`:latest`以外が指定されている: `IfNotPresent`が適用されます。

- `imagePullPolicy: Never`: イメージがローカルに存在するとみなされます。イメージをPullしようとしません。
  
{{< note >}}
  **メモ:** コンテナに同じイメージのバージョンを常に使わせるために、その[ダイジェスト](https://docs.docker.com/engine/reference/commandline/pull/#pull-an-image-by-digest-immutable-identifier) (例えば`sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2`)を指定することができます。これは指定したバージョンのイメージを一意に識別するので、ダイジェストの値を変えない限りKubernetesによって更新されることは決してありません。
{{< /note >}}

{{< note >}}
  **メモ:** 本番環境にコンテナをデプロイする場合、実行しているイメージのバージョンを追跡することが難しくなり、適切なロールバックも難しくなることから、`:latest`タグの使用は避けるべきです。
{{< /note >}}

{{< note >}}
  **メモ:** イメージプロバイダのキャッシングの仕組みは`imagePullPolicy: Always`でさえ効果的にします。例えばDockerであれば、イメージが既に存在していれば、すべてのイメージレイヤがキャッシュされていて、イメージのダウンロードが不要になるため、Pullは高速になります。
{{< /note >}}

## kubectlの利用

- `kubectl apply -f <directory>` または `kubectl create -f <directory>`を使うこと。これは `<directory>` にあるすべての `.yaml`, `.yml`, `.json`ファイルからKubernetes構成を探し、`apply`または`create`に渡す。

- `get` と `delete`操作に対しては特定のオブジェクト名のかわりにラベルセレクタを使う。[ラベルセレクタ](/ja/docs/concepts/overview/working-with-objects/labels/#ラベルセレクタ)と[効果的なラベルの使用](/ja/docs/concepts/cluster-administration/manage-deployment/#効果的なラベルの使用)のセクションを参照。

- 素早く単一コンテナのDeploymentとサービスを作るために`kubectl run`と`kubectl expose`を使う。例は[クラスタのアプリケーションにアクセスするためにサービスを使う](/docs/tasks/access-application-cluster/service-access-application-cluster/)を参照。

{{% /capture %}}
