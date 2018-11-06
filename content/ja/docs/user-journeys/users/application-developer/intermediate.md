---
layout: docsportal
css: /css/style_user_journeys.css
js: https://use.fontawesome.com/4bcc658a89.js, https://cdnjs.cloudflare.com/ajax/libs/prefixfree/1.0.7/prefixfree.min.js
title: 中級
track: "USERS › APPLICATION DEVELOPER › INTERMEDIATE"
content_template: templates/user-journey-content
---

{{% capture overview %}}

{{< note  >}}
このページでは、Kubernetesの経験者であることを仮定しています。このため、Kubernetesクラスタとの基本的なやりとりと、アプリケーションを実行するDeploymentのようなAPIオブジェクトを使う経験しておくべきです。<br><br>そうでなければ、{{< link text="初級アプリケーション開発者" url="/ja/docs/user-journeys/users/application-developer/foundational/" >}}のトピックを最初に読んでください。
{{< /note  >}}
このページとリンク先の記事をチェックすると、以下の点についてより良く理解できるでしょう。

* Deployment以外のKubernetesワークロードパターン
* Kubernetesアプリケーションを本番稼動可能にするために必要なこと
* 開発ワークフローを改善できるコミュニティツール

{{% /capture %}}

{{% capture body %}}

## さらなるワークロードパターン {#learn-additional-workload-pattern}

Kubernetesのユースケースが複雑になるにしたがって、Kubernetesが提供するツールキットをさらに習熟することが手助けとなることに気付くかもしれません。{{< glossary_tooltip text="Deployment" term_id="deployment" >}}のような{{< link text="基本のワークロード" url="/ja/docs/user-journeys/users/application-developer/foundational/#section-2" >}}オブジェクトは、アプリケーションの実行や更新、スケールを簡単にしてくれますが、すべてのシナリオに適しているわけではありません。

以下のAPIオブジェクトは、*永続* や *終了* といった、さらなるワークロードタイプに対する機能を提供します。

#### 永続ワークロード

Deploymentのように、これらのAPIオブジェクトは手動で終了させられるまで、いつまでも実行し続けます。長時間実行するアプリケーションに最適です。

*  **{{< glossary_tooltip text="StatefulSet" term_id="statefulset" >}}** - Deploymentのように、StatefulSetはある数のレプリカを実行するよう指示することができます。

    {{< note >}} Deploymentがステートフルなワークロードを扱えないというのは誤解です。{{< glossary_tooltip text="PersistentVolume" term_id="persistent-volume" >}}を使うことで、Deploymentの個々のPodのライフサイクルを超えたデータの永続化が可能です。
    {{< /note >}}

    しかしながら、StatefulSetは、Deploymentよりも「回復」のふるまいについて、より強い保証を提供できます。StatefulSetはPodに対する安定したIDを維持します。次の表はこれがどのようになるかの具体例です。

    |   | Deployment | StatefulSet |
    |---|---|---|
    | **Pod名の例** | `example-b1c4` | `example-0` |
    | **Podが死んだ時** | 新しい名前 `example-a51z` で *どこか* のノードに再スケジュールされる | `example-0` として、同じノードに再スケジュールされる |
    | **ノードが到達不能になった時** | Podは新しいノードに新しい名前でスケジュールされる | Podは"Unknown"となり、ノードオブジェクトが強制的に削除されるまで再スケジュールされない |

    実際に、これは、レプリカ (Pod) が強く一貫した動作でワークロードと協調する必要があるというシナリオにおいて、StatefulSetが最適であることを意味します。各PodのIDが保証されることは、ノードが到達不能になる場合における{{< link text="スプリットブレイン" url="https://en.wikipedia.org/wiki/Split-brain_(computing)" >}}の副作用を避ける手助けになります。これはCassandraやElasticsearchのような分散データストアに対して、StatefulSetが非常に適していることを示します。

* **{{< glossary_tooltip text="DaemonSet" term_id="daemonset" >}}** - DaemonSetはクラスタのすべてのノードで、ノードの追加や入れ換えがあったとしても、継続的に動作します。この保証は以下のようなクラスタでのグローバルなふるまいのセットアップに特に便利です。

  * `fluentd`のようなアプリケーションからのロギングとモニタリング
  * ネットワークプロキシや{{< link text="サービスメッシュ" url="https://www.linux.com/news/whats-service-mesh-and-why-do-i-need-one" >}}

#### 終了するワークロード {#terminating-workload}

Deploymentと対照に、これらのAPIオブジェクトは有限です。これらは指定された数のPodが正常に完了すれば停止します。

* **{{< glossary_tooltip text="Job" term_id="job" >}}** - これはスクリプトの実行やワークキューのセットアップのような、1回限りのタスクを実行するために使われます。これらのタスクは直列にも並列にも実行できます。Jobは並列プロセスの密な通信をサポートしないので、これらのタスクは独立であるべきです。詳細は{{< link text="Jobパターン" url="/ja/docs/concepts/workloads/controllers/jobs-run-to-completion/#job-patterns" >}}を参照してください。

* **{{< glossary_tooltip text="CronJob" term_id="cronjob" >}}** - Jobと似ていますが、指定した時間や間隔での実行をスケジュールできます。CronJobはリマインダメールの送信やバックアップジョブの実行に使えます。*crontab* と似た文法でセットアップされます。

#### その他のリソース

さらなる情報は、{{< link text="APIリファレンス" url="{{ reference_docs_url }}" >}}や{{< link text="追加のKubernetesリソースタイプ一覧" url="/docs/reference/kubectl/overview/#resource-types" >}}でチェックできます。

ここで言及していない、便利だと思える追加の機能があるかもしれません。それらは{{< link text="Kubernetesドキュメント" url="/ja/docs/home/?path=browse" >}}で取り扱います。

## 本番稼動可能なワークロードのデプロイ

{{< link text="Guestbookアプリ" url="/ja/docs/tutorials/stateless-application/guestbook/" >}}のような初級者向けチュートリアルは、ワークロードの起動に焦点をおいています。このプロトタイプはKubernetesの知識を得るにはとても良いものです。しかし、ワークロードを確実かつセキュアに本番に昇格させるには、いくつかのベストプラクティスに従う必要があります。

#### 宣言的な構成

Kubernetesクラスタとやりとりを行うために{{< glossary_tooltip text="kubectl" term_id="kubectl" >}}を使うのではないかと思います。kubectlはクラスタの現在の状態をデバッグ (ノード数のチェックなど) したり、稼働中のKubernetesオブジェクトを編集 (`kubectl scale`を使ってワークロードのレプリカ数を更新など) したりするために使われます。

Kubernetesオブジェクトを更新するためにkubectlを使う場合、異なるアプローチに対応する異なるコマンドがあることを知っておくことが重要です。

* {{< link text="純粋な命令" url="/docs/tutorials/object-management-kubectl/imperative-object-management-command/" >}}
* {{< link text="ローカルの構成ファイルを使った命令" url="/docs/tutorials/object-management-kubectl/imperative-object-management-configuration/" >}} (通常はYAML)
* {{< link text="ローカルの構成ファイルを使った宣言" url="/docs/tutorials/object-management-kubectl/declarative-object-management-configuration/" >}} (通常はYAML)

それぞれのアプローチには利点と欠点がありますが、(`kubectl apply -f`のような) 宣言的なアプローチが本番では最も便利です。このアプローチだと、必要とする状態についての信頼できる情報源として、ローカルのYAMLファイルを頼ることになります。これは、構成のバージョン管理を可能にし、コードレビューと監査の追跡に有用です。

構成についての追加のベストプラクティスは{{< link text="このガイド" url="/docs/concepts/configuration/overview/" >}}を参照してください。

#### セキュリティ {#security}

*最小権限の原則* をご存じかもしれません。ソフトウェアを書いたり使ったりする場合の権限が広すぎると、情報漏洩の危険が制御できないほど増大します。OSでソフトウェアへの`sudo`権限を注意深く扱っていますか？そうであるのなら、{{< glossary_tooltip text="Kubernetes API" term_id="kubernetes-api" >}}サーバへのワークロード権限の付与も同じように注意深くあるべきです。APIサーバはクラスタの信頼できる情報源へのゲートウェイです。これはクラスタ状態の読み込みを変更を行うためのエンドポイントを提供します。

{{< glossary_tooltip text="クラスタ管理者" term_id="cluster-operator" >}}は以下のようにしてAPIアクセスを制御できます。

* **{{< glossary_tooltip text="ServiceAccounts" term_id="service-account" >}}** - Podとむすびつくことのできる"ID"
* **{{< glossary_tooltip text="RBAC" term_id="rbac" >}}** - ServiceAccountに明示的に権限を与える方法の1つ

セキュリティベストプラクティスについてのより総合的な記事については以下を参照してください。

* {{< link text="認証" url="/ja/docs/reference/access-authn-authz/authentication/" >}} (名乗っている通りのユーザかどうか？)
* {{< link text="認可" url="/ja/docs/reference/access-authn-authz/authorization/" >}} (要求する動作への権限を持っているかどうか？)

#### リソース分離と管理 {#resource-isolation-and-management}

ワークロードが複数のチームやプロジェクトからなる *マルチテナント* 環境で運用する場合、必ずしもコンテナはノードで単独で実行しません。他のコンテナとノードリソースを共有します。

別のクラスタ管理者がクラスタを管理していたとしても、以下を知っておくことは有用です。

* **{{< glossary_tooltip text="名前空間" term_id="namespace" >}}**: 分離のために使います
* **{{< link text="リソースクオータ" url="/docs/concepts/policy/resource-quotas/" >}}**: チームのワークロードが使えるものに影響します
* **{{< link text="メモリ" url="/docs/tasks/configure-pod-container/assign-memory-resource/" >}}と{{< link text="CPU" url="/docs/tasks/configure-pod-container/assign-cpu-resource/" >}}要求**: Podやコンテナに与えられます
* **{{< link text="モニタリング" url="/docs/tasks/debug-application-cluster/resource-usage-monitoring/" >}}**: クラスタレベルとアプリレベルの両方です

このリストは完全に網羅するものではありませんが、多くのチームはこれらすべてをケアする既存のプロセスを持っています。そうでない場合には、このKubernetesドキュメントが細部にわたって非常に豊かであることがお分かりになると思います。

## ツールで開発ワークフローを改善する {#improve-your-dev-workflow-with-tooling}

アプリケーション開発者として、ワークフローの中で次のようなツールに出会うと思われます。

#### kubectl

`kubectl`はKubernetesクラスタを容易に読み込み、編集できるようにするコマンドラインツールです。アプリケーションのスケーリングやノード情報の取得のような、よく使う操作に対する便利で短いコマンドを提供します。kubectlはどのように動作しているのでしょう？基本的にはAPIリクエストを作成するラッパーでしかありません。Kubernetes APIのGoライブラリである{{< link text="client-go" url="https://github.com/kubernetes/client-go/#client-go" >}}を使って書かれています。

最もよく使われるkubectlコマンドについては、{{< link text="kubectl cheatsheet" url="/docs/reference/kubectl/cheatsheet/" >}}を参照してください。これは次のようなトピックを扱います。

* {{< link text="kubeconfigファイル" url="/docs/tasks/access-application-cluster/configure-access-multiple-clusters/" >}} - kubeconfigファイルはkubectlにどのクラスタとやりとりするのかを教えます。(開発と本番のような) 複数のクラスタを参照することができます。
* {{< link text="さまざまな出力形式が利用可能" url="/docs/reference/kubectl/cheatsheet/#formatting-output" >}} - あるAPIオブジェクトについての情報を列挙するために`kubectl get`を使う場合に知っておくと便利です。

* {{< link text="JSONPath出力形式" url="/docs/reference/kubectl/jsonpath/" >}} - これは上の出力形式と関係します。JSONPathは、`kubectl get`の出力の特定のサブフィールド ({{< glossary_tooltip text="Service" term_id="service" >}}のURLなど) をパースする際に特に便利です。

* {{< link text="`kubectl run` vs `kubectl apply`" url="/docs/reference/kubectl/conventions/" >}} - これは前のセクションの[宣言的構成](#declarative-configuration)に直結します。

kubectlコマンドとそのオプションの完全なリストは、{{< link text="リファレンスガイド" url="/docs/reference/generated/kubectl/kubectl-commands" >}}を参照してください。

#### Helm

コミュニティによって事前にパッケージ化された構成を利用するために、**{{< glossary_tooltip text="Helm chart" term_id="helm-chart" >}}**が使えます。

Helm chartはJenkinsやPostgresのような特定のアプリのYAML構成をパッケージ化しています。そのため、最小の追加構成でクラスタにこれらのアプリをインストールして実行できます。このアプローチはカスタム実装ロジックを必要としない「既製」コンポーネントに最も有用です。

自身のKubernetesアプリ構成を書くことについては、{{< link text="thriving ecosystem of tools" url="https://docs.google.com/a/heptio.com/spreadsheets/d/1FCgqz1Ci7_VCz_wdh8vBitZ3giBtac_H8SBw4uxnrsE/edit?usp=drive_web" >}}が参考になるかもしれません。

## リソースの探索 {#explore-additional-resources}

#### リファレンス {#references}

今や、Kubernetesについてかなり詳しくなったので、以下のリファレンスページを見ることが便利であると気付くと思います。そうすることで、存在する他の機能についての高いレベルの視点を得られます。

* {{< link text="よく使われる`kubectl`コマンド" url="/docs/reference/kubectl/cheatsheet/" >}}
* {{< link text="Kubernetes APIリファレンス" url="{{ reference_docs_url }}" >}}
* {{< link text="用語集" url="/docs/reference/glossary/" >}}

加えて、{{< link text="the Kubernetes Blog" url="https://kubernetes.io/blog/" >}}にもKubernetesのデザインパターンやケーススタディについての有用な記事がたびたび投稿されます。

#### What's next

このページの内容がかなり易しいと感じられ、さらに学びたいと思うのであれば、以下のユーザジャーニーをチェックしてください。

* {{< link text="上級アプリケーション技術者" url="/docs/user-journeys/users/application-developer/advanced/" >}} - さらに深い、このジャーニーの次のレベルです。
* {{< link text="初級クラスタ管理者" url="/docs/user-journeys/users/cluster-operator/foundational/" >}} - 他のジャーニーを探索することで、見識を広げます。
{{% /capture %}}
