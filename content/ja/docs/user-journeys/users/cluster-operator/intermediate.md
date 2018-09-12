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

{{% /capture %}}
