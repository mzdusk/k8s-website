---
content_template: templates/concept
title: リソース監視のツール
---

{{% capture overview %}}

アプリケーションをスケールし、信頼性の高いサービスを提供するために、アプリケーションがデプロイされる時に
どうふるまうのかを理解する必要があります。コンテナや[Pod](/docs/concepts/workloads/pods/pod/)、
[Service](/docs/concepts/services-networking/service/)、さらにクラスタ全体の特徴を調べることで、Kubernetesクラスタでの
アプリケーションのパフォーマンスを調べることができます。Kubernetesはアプリケーションのリソース使用量の
詳細な情報を提供します。この情報によって、アプリケーションのパフォーマンスと、全体のパフォーマンスを
改善するために削除できるボトルネックがどこにあるのかを評価できます。

{{% /capture %}}

{{% capture body %}}

Kubernetesでは、アプリケーションの監視は単一の監視ソリューションに依存しません。新しいクラスタ上で、
監視の統計情報を収集する2つの分離したパイプラインをデフォルトで使うことができます。

- [**リソース・メトリクス・パイプライン**](#resource-metrics-pipeline)はHorizontalPodAutoscalerコントローラのような
  クラスタ・コンポーネントに関連する限定されたメトリクスのセットを`kubectl top`ユーティリティと同じように提供します。
  これらのメトリクスは[metrics-server](https://github.com/kubernetes-incubator/metrics-server)で制御され、
  `metrics.k8s.io`経由で公開されます。`metrics-server`はクラスタのすべてのノードを探し、各ノードの
  [Kubelet](/docs/reference/command-line-tools-reference/kubelet/)にCPUとメモリの使用量を問い合わせます。Kubeletは
  [cAdvisor](https://github.com/google/cadvisor)からデータを取り出します。`metrics-server`は軽量な短期間インメモリ
  ストアです。

- [**フル・メトリクス・パイプライン**](#full-metrics-pipelines)は、Prometheusのように、よりリッチなメトリクスに
  アクセスできるようにします。加えて、Kubernetesは、HorizontalPodAutoscalerのような仕組みを使って、自動的に
  スケーリングしたりクラスタの状態を変化させることで、これらのメトリクスに反応することができます。この監視パイプ
  ラインはKubeletからメトリクスを取り出し、`custom.metrics.k8s.io`や`external.metrics.k8s.io`で実装された
  アダプタ経由でKubernetesに公開します。

## リソース・メトリクス・パイプライン {#resource-metrics-pipeline}

### Kubelet

KubeletはKubernetsマスタとノードを橋渡しします。マシン上で稼動するPodとコンテナを管理します。Kubeletは
各Podを構成要素であるコンテナに変換し、cAdvisorから各コンテナの使用量の統計を取得します。そして、
集計されたPodのリソース使用量をREST API経由で公開します。

### cAdvisor

cAdvisorはコンテナリソース使用量とパフォーマンスを分析するオープンソースのエージェントです。これはコンテナ専用で、
Dockerコンテナをネイティブにサポートします。Kubernetesでは、cAdvisorはKubeletバイナリに統合されています。
cAdvisorはマシン上のすべてのコンテナを自動で探し、CPU、メモリ、ファイルシステム、ネットワークの使用量の統計を
収集します。cAdvisorは'root'コンテナを分析することで、マシン全体の使用量も提供します。

ほとんどのKubernetesクラスタでは、cAdvisorはマシン上のコンテナの4194ポートから単純なUIを提供します。
マシン全体の使用量を示すcAdvisorのUIの一部のスクリーンショットを示します。

![cAdvisor](/images/docs/cadvisor.png)

## フル・メトリクス・パイプライン {#full-metrics-pipelines}

Kubernetes向けフル・メトリクス・パイプラインのソリューションが数多くあります。

### Prometheus

[Prometheus](https://prometheus.io) はkubernetes、ノード、prometheus自身をネイティブに監視できます。
[Prometheus Operator](https://coreos.com/operators/prometheus/docs/latest/)はKubernetes上での
Prometheusのセットアップを簡単にしてくれ、
[Prometheus adapter](https://github.com/directxman12/k8s-prometheus-adapter)を使ってカスタム・メトリクス API
を提供できるようにします。Prometheusは堅牢なクエリ言語と、データの問い合わせと可視化を行う組み込みダッシュボード
を提供します。Prometheusは[Grafana](https://prometheus.io/docs/visualization/grafana/)向けのデータソースも
提供します。

### Google Cloud Monitoring

Google Cloud Monitoringは、アプリケーションの重要なメトリクスを可視化したりアラートを上げたりするのに使う
ホストされた監視サービスです。Kubernetesからメトリクスを収集することができ、
[Cloud Monitoring Console](https://app.google.stackdriver.com/)を使ってアクセスできます。Kubernetesクラスタから
集めた情報を可視化するダッシュボードの作成とカスタマイズができます。

この動画はHeapsterを使ったGoogle Cloud Monitoringの構成と実行の方法を示します。

[![how to setup and run a Google Cloud Monitoring backed Heapster](https://img.youtube.com/vi/xSMNR2fcoLs/0.jpg)](https://www.youtube.com/watch?v=xSMNR2fcoLs)

{{< figure src="/images/docs/gcm.png" alt="Google Cloud Monitoringダッシュボードの例" title="Google Cloud Monitoringダッシュボードの例" caption="このダッシュボードはクラスタ全体のリソース使用量を表示しています。" >}}

## CronJob監視 {#cronjob-monitoring}

### Kubernetes Job Monitor

[Kubernetes Job Monitor](https://github.com/pietervogelaar/kubernetes-job-monitor)ダッシュボードを使うと、
クラスタ管理者はどのジョブが実行しているのかや完了したジョブのステータスを見ることができます。

### New Relic Kubernetesによる監視の統合 {#new-relic-kubernetes-monitoring-integration}

[New Relic Kubernetes](https://docs.newrelic.com/docs/integrations/host-integrations/host-integrations-list/kubernetes-monitoring-integration)統合は
Kubernetes環境のパフォーマンスの可視性を強化します。New RelicのKubernetes統合は、Kubernetsオブジェクトからのメトリクスを
報告することで、コンテナ・オーケストレーション・レイヤを編集します。この統合はKubernetesノードや名前空間、Deployment、
ReplicaSet、Pod、コンテナについての洞察を与えてくれます。

特徴:
Kubernetes環境についての洞察を素早く得るために、事前に構築されたダッシュボードでデータを見ることができます。
自動的に報告されたデータから、洞察のカスタムクエリとグラフを作成できます。
Kubernetsデータのアラート条件を作成できます。
詳細はこの[ページ](https://docs.newrelic.com/docs/integrations/host-integrations/host-integrations-list/kubernetes-monitoring-integration)を
参照してください。

{{% /capture %}}
