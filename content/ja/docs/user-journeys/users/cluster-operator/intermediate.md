---
layout: docsportal
css: /css/style_user_journeys.css
js: https://use.fontawesome.com/4bcc658a89.js, https://cdnjs.cloudflare.com/ajax/libs/prefixfree/1.0.7/prefixfree.min.js
title: Intermediate
track: "USERS > CLUSTER OPERATOR > INTERMEDIATE"
content_template: templates/user-journey-content
---

{{% capture overview %}}

もし、Kubernetes の理解を広げることに関心のあるクラスタ管理者であれば、このページとそのリンク先の話題は、[クラスタ管理者基礎のページ](/ja/docs/user-journeys/users/cluster-operator/foundational)で提供される情報を拡張します。このページから、完全な本番クラスタを管理するために必要とされる重要な Kubernetes のタスクについての情報を得ることができます。

{{% /capture %}}

{{% capture body %}}

## Ingress, ネットワーキング, ストレージ, ワークロードを使う

Kubernetes の導入ではシンプルなステートレスアプリケーションについて議論します。より複雑な開発やテスト本番環境に進むには、より複雑なケースを考慮する必要があります。

通信: Ingress とネットワーキング

* [Ingress](/ja/docs/concepts/services-networking/ingress/)

ストレージ: Volumes と PersistentVolumes

* [Volumes](/ja/docs/concepts/storage/volumes/)
* [Persistent Volumes](/ja/docs/concepts/storage/persistent-volumes/)

ワークロード

* [DaemonSets](/ja/docs/concepts/workloads/controllers/daemonset/)
* [Stateful Sets](/ja/docs/concepts/workloads/controllers/statefulset/)
* [Jobs](/ja/docs/concepts/workloads/controllers/jobs-run-to-completion/)
* [CronJobs](/ja/docs/concepts/workloads/controllers/cron-jobs/)

Pod

* [Pod のライフサイクル](/ja/docs/concepts/workloads/pods/pod-lifecycle/)
  * [Init コンテナ](/ja/docs/concepts/workloads/pods/init-containers/)
  * [Pod Presets](/ja/docs/concepts/workloads/pods/podpreset/)
  * [コンテナライフサイクルフック](/ja/docs/concepts/containers/container-lifecycle-hooks/)

Podがどのようにスケジューリングされ、優先順位付けされ、崩壊するか

* [汚染と許容](/ja/docs/concepts/configuration/taint-and-toleration/)
* [Pod と優先順位](/ja/docs/concepts/configuration/pod-priority-preemption/)
* [崩壊](/ja/docs/concepts/workloads/pods/disruptions/)
* [Pod のノードへの割り当て](/ja/docs/concepts/configuration/assign-pod-node/)
* [コンテナに対する計算資源の割り当て](/ja/docs/concepts/configuration/manage-compute-resources-container/)
* [構成ベストプラクティス](/ja/docs/concepts/configuration/overview/)

## セキュリティベストプラクティスの実践

クラスタをセキュアにすることはKubernetes自身のスコープ外の作業も含みます。

Kubernetesではアクセス制御を構成します。

* [Kubernetes APIへのアクセス制御](/ja/docs/reference/access-authn-authz/controlling-access/)
* [認証](/ja/docs/reference/access-authn-authz/authentication/)
* [アドミッションコントローラの使用](/ja/docs/reference/access-authn-authz/admission-controllers/)

認可も構成します。したがって、どのようにユーザとサービスがAPIサーバに認証し、もしくはアクセス権があるかどうかだけでなく、どのリソースへのアクセス権があるのかも決定します。ロールベースアクセス制御 (RBAC) はKubernetesリソースへの認可の制御のための推奨される仕組みです。

* [認可の概要](/ja/docs/reference/access-authn-authz/authorization/)
* [RBAC認可の使用](/ja/docs/reference/access-authn-authz/rbac/)

パスワードやトークン、キーといった機微なデータを保持するためにSecretを作成すべきです。しかしながら、Secretが提供できる保護には限界があることに留意してください。[Secretのドキュメントのリスクセクション](/ja/docs/concepts/configuration/secret/#risks)を参照してください。

## カスタムロギングとモニタリングの実装

クラスタの状態をモニタリングすることは重要です。メトリクスを集め、ロギングし、それらの情報へのアクセスを提供することは共通のニーズです。Kubernetesはいくつかの基本的なロギング構造を提供しますが、ログデータの集約と分析を手助けする追加のツールを使いたくなるかもしれまん。

どのようにしてコンテナがロギングするのかと一般的なパターンを理解するために、[Kubernetesロギングの基本](/docs/concepts/cluster-administration/logging/)から始めましょう。クラスタ管理者はこれらのログを集めて集約するための何かを追加したくなることがしばしばあります。以下のトピックを参照してください。

* [ElasticsearchとKibanaを使ったロギング](/docs/tasks/debug-application-cluster/logging-elasticsearch-kibana/)
* [Stackdriverを使ったロギング](/docs/tasks/debug-application-cluster/logging-stackdriver/)

ログの集約のように、多くのクラスタは、メトリクスの取得とその閲覧を手助けする追加のソフトを活用します。ツールの概要が[計算資源、ストレージ資源、ネットワーク資源をモニタリングするためのツール](/docs/tasks/debug-application-cluster/resource-usage-monitoring/)にあります。Kubenetesもカスタムメトリクスを使ったHorizontal Pos Autoscalerによって使われる[コアメトリクスパイプライン](/docs/tasks/debug-application-cluster/core-metrics-pipeline/)をサポートしています。

他のCNCFプロジェクトである[Prometheus](https://prometheus.io/)はメトリクスのキャプチャと一時的な収集をサポートする一般的な選択肢です。インストールには、[stable/prometheus](https://github.com/kubernetes/charts/tree/master/stable/prometheus) [helm](https://helm.sh/) chartや、CoreOSが提供する[prometheus operator](https://github.com/coreos/prometheus-operator)と[kube-prometheus](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus)などのオプションがあり、これらにはGraphanaダッシュボードと一般的な構成がアドオンされています。

[Minikube](https://github.com/kubernetes/minikube)やいくつかのKubernetesクラスタでの一般的な構成は
、[InfluxDBとGraphanaと連携した](https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md)[Heapster](https://github.com/kubernetes/heapster)を使います。[この構成でのインストール方法のウォークスルー](https://blog.kublr.com/how-to-utilize-the-heapster-influxdb-grafana-stack-in-kubernetes-for-monitoring-pods-4a553f4d36c9)があります。1.11以降、[sig-instrumentation](https://github.com/kubernetes/community/tree/master/sig-instrumentation)のとおり、Heapsterは非推奨となりました。詳細は[Prometheus vs. Heapster vs. Kubernetes Metrics APIs](https://brancz.com/2018/01/05/prometheus-vs-heapster-vs-kubernetes-metrics-apis/)を参照してください。

[Datadog](https://docs.datadoghq.com/integrations/kubernetes/)のようなホステッドデータ分析サービスもKubernetesとの統合を提案しています。

## 追加のリソース

クラスタ管理:

* [Clusterのトラブルシューティング](/docs/tasks/debug-application-cluster/debug-cluster/)
* [PodとReplication Controllerのデバッグ](/docs/tasks/debug-application-cluster/debug-pod-replication-controller/)
* [Initコンテナのデバッグ](/docs/tasks/debug-application-cluster/debug-init-containers/)
* [Stateful Setのデバッグ](/docs/tasks/debug-application-cluster/debug-stateful-set/)
* [アプリケーションのデバッグ](/docs/tasks/debug-application-cluster/debug-application/)
* [クラスタ調査のためにexplorerを使う](https://github.com/kubernetes/examples/blob/master/staging/explorer/README.md)

{{% /capture %}}
