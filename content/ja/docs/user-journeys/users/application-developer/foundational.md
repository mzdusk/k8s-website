---
layout: docsportal
css: /css/style_user_journeys.css
js: https://use.fontawesome.com/4bcc658a89.js, https://cdnjs.cloudflare.com/ajax/libs/prefixfree/1.0.7/prefixfree.min.js
title: Foundational
track: "USERS › APPLICATION DEVELOPER › FOUNDATIONAL"
content_template: templates/user-journey-content
---

{{% capture overview %}}
Kubernetesでアプリケーションを実行することに関心のある開発者であれば、このページとリンク先のトピックが基礎から始めることを手助けします。このページではまず開発ワークフローを述べますが、{{< link text="中級" url="/docs/home/?path=users&persona=app-developer&level=intermediate" >}}ではより高度な本番セットアップを取りあげます。

{{< note  >}}
**メモ**<br>このアプリ開発者の「ユーザジャーニー」はKubernetesの包括的な概要では *ありません* 。ここでは基礎のインフラが *どのように* 動作するかではなく、 *何を* Kubernetesで開発、テスト、デプロイするのかに焦点をあてます。<br><br>多くの組織ではひとりで両方を管理することもありますが、一般的に{{< glossary_tooltip text="クラスタ管理者" term_id="cluster-operator" >}}は専任になります。
{{< /note  >}}
{{% /capture %}}


{{% capture body %}}

## クラスタの用意

#### Webベースの環境

Kubernetesに触れたばかりで、完全な開発環境のセットアップなしに単に経験したいだけであれば、 *Webベース環境* が適しています。

* {{< link text="Kubernetesの基本" url="/docs/tutorials/kubernetes-basics/#basics-modules" >}} - 6つの基本的なKubernetesのワークフローを紹介します。各セクションはブラウザベースでの説明を行い、インタラクティブな演習はそれ自身のKubernetes環境を備えています。

* {{< link text="Katacoda" url="https://www.katacoda.com/courses/kubernetes/playground" >}} - 上記の *Kubernetesの基本* で使われる環境と等価な遊び場です。Katacodaは、"Liveness and Readiness Healthchecks"のような、{{< link text="高度なチュートリアル" url="https://www.katacoda.com/courses/kubernetes/" >}}も提供しています。

* *(オプション)* ローカルの開発環境の一部としてMinikubeクラスタを実行するつもりであれば、{{< link text="Dockerをインストール" url="/docs/setup/independent/install-kubeadm/#installing-docker" >}}してください。

   MinikubeにはDockerデーモンが含まれていますが、ローカルでアプリケーションを開発しているのなら、ワークフローをサポートするための独立したDockerインスタンスが必要になってきます。これは{{< glossary_tooltip text="コンテナ" term_id="container" >}}の作成とコンテナレジストリへのPushをできるようにします。

   {{< note  >}}
   Kubernetesとの完全な互換性のために、バージョン1.12を推奨しますが、別のいくつかのバージョンでもテストされ、動作が確認されています。
   {{< /note  >}}

クラスタについての基本的な情報は`kubectl cluster-info`や`kubectl get nodes`で取得できます。しかしながら、実際に起こっていることを知るには、アプリケーションをクラスタにデプロイする必要があります。これは次のセクションで取り上げます。

## アプリケーションのデプロイ

#### 基本のワークロード

以下の例はKubernetesアプリのデプロイの基本を示します。

  * **ステートレスアプリ**: {{< link text="シンプルなNginxサーバのデプロイ" url="/ja/docs/tasks/run-application/run-stateless-application-deployment/" >}}
  * **ステートフルアプリ**: {{< link text="MySQLデータベースのデプロイ" url="/docs/tasks/run-application/run-single-instance-stateful-application/" >}}

これらのデプロイタスクを通じて、以下に詳しくなれます。

* 全体的な概念

  * **構成ファイル** - YAMLまたはJSONで書かれ、Kubernetes APIオブジェクトの観点からアプリケーションの望ましい状態を記述します。ファイルには1つまたは複数のAPIオブジェクトの記述を含められます (*マニフェス*)。(ステートレスアプリの[YAMLの例](/ja/docs/tasks/run-application/run-stateless-application-deployment/#creating-and-exploring-an-nginx-deployment)を参照してください)

  * **{{< glossary_tooltip text="Pod" term_id="pod" >}}** - Kubernetesで実行するすべてのワークロードの基本単位です。*Deployment* や *Job* といったワークロードには1つまたは複数のPodで構成されます。詳細は{{< link text="Podとノードの説明" url="/docs/tutorials/kubernetes-basics/explore-intro/" >}}を参照してください。

* 一般的なワークロードオブジェクト
  * **{{< glossary_tooltip text="Deployment" term_id="deployment" >}}** - *X* 個のコピー (Pod) を実行する最も一般的な方法です。コンテナイメージのローリングアップデートをサポートします。

  * **{{< glossary_tooltip text="サービス" term_id="service" >}}** - Deploymentはそれ自身でトラフィックを受け取ることができません。サービスのセットアップはDeploymentへのリクエストの取得とロードバランスを構成する最もシンプルな方法の1つです。利用するサービスの`type`によって、これらのリクエストを外部のクライアントアプリから受けとったり、同じクラスタ内のアプリに制限したりできます。サービスは{{< glossary_tooltip text="ラベル" term_id="label" >}}選択を使って特定のDeploymentと結びつけられます。

以降のトピックも基本的なアプリケーションのデプロイについて知るのに有用です。

#### メタデータ

キーバリューフィールドを付与することで、Kubernete APIオブジェクトについてのカスタム情報を指定することもできます。2つの方法があります。

* **{{< glossary_tooltip text="ラベル" term_id="label" >}}** - APIオブジェクトの分類と選択に使える識別メタデータです。以下のように、ラベルにはさまざまな活用があります。

  * *Deploymentで実行する正しい数のレプリカ (Pod) を保つ。* 指定されたラベル ({{< link text="ステートレスアプリの例" url="/ja/docs/tasks/run-application/run-stateless-application-deployment/#creating-and-exploring-an-nginx-deployment" >}}での`app: nginx`)はDeploymentが新しく作成したPodに (`spec.template.labels`フィールドの値として) 印をつける、どのPodが既に管理されているかを (`spec.selector.matchLabels`の値として) 問い合わせるために使います。
  
  * *サービスをDeploymentに結びつける。* `selector`フィールドを使います。これは{{< link text="ステートフルアプリの例" url="/docs/tasks/run-application/run-single-instance-stateful-application/#deploy-mysql" >}}で例示します。
  
  * *{{< glossary_tooltip text="kubectl" term_id="kubectl" >}}を使う場合に、Kubernetesオブジェクトのサブセットを探す。* 例えば、`kubectl get deployments --selector=app=nginx`はnginxアプリのDeploymentのみを表示します。

* **{{< glossary_tooltip text="アノテーション" term_id="annotation" >}}** - APIオブジェクトに付与できる非識別メタデータで、通常、分類目的で使用することはありません。これはGit のSHAや、PR番号、観測ダッシュボードへのURLポインタといった、アプリケーションのデプロイについての補足情報をたびたび提供します。

#### ストレージ

ストレージについても考えたいと思うでしょう。Kubernetesは様々なストレージのニーズに応じて、様々なタイプのストレージAPIオブジェクトを提供しています。

* **{{< glossary_tooltip text="Volumes" term_id="volume" >}}** - Podのライフサイクルに結びつうストレージを定義します。したがって、コンテナストレージよりも永続的です。詳細は{{< link text="Volumeストレージの構成方法" url="/docs/tasks/configure-pod-container/configure-volume-storage/" >}}や{{< link text="Volumeストレージの詳細" url="/ja/docs/concepts/storage/volumes/" >}}を参照してください。

* **{{< glossary_tooltip text="PersistentVolumes" term_id="persistent-volume" >}}** と **{{< glossary_tooltip text="PersistentVolumeClaims" term_id="persistent-volume-claim" >}}** - クラスタレベルのストレージを定義します。通常、クラスタ管理者がPersistentVolumeオブジェクトを定義し、クラスタユーザ (アプリケーション開発者) がアプリケーションが必要とするPersistentVolumeClaimオブジェクトを定義します。詳細は{{< link text="永続ストレージのセットアップ方法" url="/docs/tasks/configure-pod-container/configure-persistent-volume-storage/" >}}や{{< link text="PersistentVolumeの詳細" url="/ja/docs/concepts/storage/persistent-volumes/" >}}を参照してください。


#### 構成

必要のないコンテナイメージのリビルドを避けるため、アプリケーションの *構成データ* と実行に必要なコードは分離すべきです。これを実現する方法が2つあり、ユースケースに応じて選択すべきです。

<table>
  <thead>
    <tr>
      <th>アプローチ</th>
      <th>データのタイプ</th>
      <th>マウント方法</th>
      <th>例</th>
    </tr>
  </thead>
  <tr>
    <td><a href="/docs/tasks/inject-data-application/define-environment-variable-container/">マニフェストのコンテナ定義を使う</a></td>
    <td>非機密情報</td>
    <td>環境変数</td>
    <td>コマンドラインフラグ</td>
  </tr>
  <tr>
    <td><b>{{< glossary_tooltip text="ConfigMap" term_id="configmap" >}}</b>を使う</td>
    <td>非機密情報</td>
    <td>環境変数 または ローカルファイル</td>
    <td>nginxの構成</td>
  </tr>
  <tr>
    <td><b>{{< glossary_tooltip text="Secret" term_id="secret" >}}</b>を使う</td>
    <td>機密情報</td>
    <td>環境変数 または ローカルファイル</td>
    <td>データベースの資格情報</td>
  </tr>
</table>

{{< note  >}}
非公開にしたいデータがあるのなら、Secretを使うべきです。さもないと、悪意のあるユーザへの露出からデータを守る術がありません。
{{< /note  >}}

## 基本的なKubernetesアーキテクチャの理解

アプリケーション開発者として、Kubernetesの内部的な動作をすべて知る必要はありませんが、高い水準で理解しておくことが有用だと気付くことがあるかもしれません。

#### Kubernetesが提供するもの

普通のRailsアプリケーションをデプロイしているチームがあるとします。何らかの計算を行って、外部トラフィックを扱うためには5つのインスタンスを実行しておく必要があると決定しました。

Kubernetesや同様の自動化システムを使わない場合、以下ようなシナリオが考えられます。

{{< note >}}
1. アプリのインスタンス (完全なマシンインスタンスまたは単なるコンテナ) が1つダウンする</li>

1. チームがモニタリングをセットアップしていたので、これが連絡をしてくる</li>

1. 連絡を受けた人が中に入り、調査して、新しいインスタンスを手動で立ち上げる</li>

1. DNS/ネットワークの扱いによって、連絡を受けた人は、古いIPから新しいRailsインスタンスのIPへサービス検出メカニズムを更新する必要もあるかもしれません。
{{</ note >}}

このプロセスは退屈で不便でもあります。特に、(2)が朝早くに発生した場合には!

**しかしながら、もしKubernetsがセットアップされていれば、手動での介入は必要ありません。** クラスタのマスタノードで実行される、Kubernetesの{{< link text="コントロールプレーン" url="/docs/concepts/overview/components/#master-components" >}}は丁寧に(3)と(4)処理します。そのため、Kubernetesはたびたび *自己回復* システムと呼ばれます。

#### Kubernetes APIサーバ

Kubernetesを便利にするため、*どのような* クラスタ状態を維持したいかを知る必要があります。YAMLまたはJSONの *構成ファイル* は、この望ましい状態を1つまたは複数のAPIオブジェクト ({{< glossary_tooltip text="Deployment" term_id="deployment" >}}など) の単位で宣言します。クラスタ状態を更新させるため、これらのファイルを{{< glossary_tooltip text="Kubernetes API" term_id="kubernetes-api" >}}サーバ (`kube-apiserver`) に提示します。

状態の例には以下を含むが、これらに限定されません。

* 実行するアプリケーションやその他のワークロード
* アプリケーションとワークロードに対するコンテナイメージ
* ネットワークとディスクリソースの割り当て

APIサーバは単なるゲートウェイに過ぎず、オブジェクトデータは実際には{{< link text="*etcd*" url="https://github.com/coreos/etcd" >}}と呼ばれる高可用データストアに格納されていることに注意してください。しかしながら、ほとんどの意図と目的に対してはAPIサーバに集中することができます。ほとんどのクラスタ状態の読み書きはAPIリクエストとして実行されます。

Kubernetes APIの詳細は{{< link text="ここ" url="/docs/concepts/overview/working-with-objects/kubernetes-objects/" >}}を参照してください。

#### コントローラ

Kubernetes APIを通じて望ましい状態を宣言すれば、 *コントローラ* がクラスタの現在の状態を望ましい状態にマッチさせます。

標準的なコントローラプロセスは{{< link text="`kube-controller-manager`" url="/docs/reference/generated/kube-controller-manager/" >}}と{{< link text="`cloud-controller-manager`" url="/docs/concepts/overview/components/#cloud-controller-manager" >}}ですが、自身のコントローラも同様に書くこともできます。

これらのコントローラはすべて *制御ループ* を実装しています。簡単のため、これは以下のように考えてください。

{{< note >}}
1. クラスタの現在の状態 (X) は何？

1. 望ましいクラスタの状態 (Y) は何？

1. X == Y ?

   * `true` - 何もしない.
   * `false` - コンテナの起動や再起動、アプリケーションのレプリカ数のスケーリングなどによって、Yにするタスクを実行する。1に戻る。
{{< /note >}}

継続的なループによって、これらのコントローラは、確実にクラスタが新しい更新を拾い、望ましい状態から離れることを回避するようにします。これらのアイデアの詳細は{{< link text="ここ" url="https://kubernetes.io/docs/concepts/" >}}で取り上げます。

## 追加リソース

Kubernetesドキュメントは非常に詳細です。より深く掘り下げるのに役立つリストを挙げます。

### 基本概念

* {{< link text="Kubernetesを実行するコンポネントの詳細" url="/docs/concepts/overview/components/" >}}

* {{< link text="Kubernetesオブジェクトの理解" url="/docs/concepts/overview/working-with-objects/kubernetes-objects/" >}}

* {{< link text="ノードオブジェクトの詳細" url="/docs/concepts/architecture/nodes/" >}}

* {{< link text="Podオブジェクトの詳細" url="/docs/concepts/workloads/pods/pod-overview/" >}}

### チュートリアル

* {{< link text="Kubernetesの基本" url="/docs/tutorials/kubernetes-basics/" >}}

* {{< link text="Hello Minikube" url="/docs/tutorials/stateless-application/hello-minikube/" >}} *(Macのみ)*

* {{< link text="Kubernetes 101" url="/docs/user-guide/walkthrough/" >}}

* {{< link text="Kubernetes 201" url="/docs/user-guide/walkthrough/k8s201/" >}}

* {{< link text="Kubernetesオブジェクト管理" url="/docs/tutorials/object-management-kubectl/object-management/" >}}

### What's next

このページの話題がかなり容易だと感じ、詳細を学びたいのであれば、以下のユーザジャーニーをチェックしてください。

* {{< link text="アプリケーション開発者 中級" url="/docs/user-journeys/users/application-developer/intermediate/" >}} - このジャーニーの次のレベルに飛び込みます。
* {{< link text="Foundational Cluster Operator" url="/docs/user-journeys/users/cluster-operator/foundational/" >}} - 他のジャーニーを探索することで、知識の幅を広げます。

{{% /capture %}}
