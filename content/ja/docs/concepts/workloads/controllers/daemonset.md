---
title: DaemonSet
content_template: templates/concept
weight: 50
---

{{% capture overview %}}

_DaemonSet_は全ての(もしくはいくつかの)ノードがPodのコピーを実行することを保証します。ノードがクラスタに追加されると、Podもそこに追加されます。クラスタからノードが削除されると、それらのPodは回収されます。DaemonSetを削除することで、作られたPodはクリーンアップされます。

DaemonSetの典型的な使い方を示します。

- 各ノードで`glusterd`や`ceph`のようなクラスタストレージデーモンを実行する
- 全ノードで`fluentd`や`logstash`のようなログ収集デーモンを実行する
- 全ノードで[Promethus Node Exporter](https://github.com/prometheus/node_exporter)や`collectd`, Datadog agent, New Relic agent, Ganglia `gmond`のようなノード監視デーモンを実行する

シンプルなケースでは、すべてのノードをカバーする1つのDaemonSetが各タイプのデーモンに対して使われます。さらに複雑なセットアップでは1つのタイプのデーモンに対して、異なるオプションやハードウェアのタイプに応じた異なるメモリとCPUの要求を行う複数のDaemonSetを使います。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## DaemonSetスペックを書く

### DaemonSetを作成する

DaemonSetはYAMLファイルで記述できます。例えば、下の`daemonset.yaml`はfluentd-elasticsearch Dockerイメージを実行するDaemonSetを記述します。

{{< code file="daemonset.yaml" >}}

* このYAMLファイルに基いたDaemonSetを作成する

```
kubectl create -f daemonset.yaml
```

### 必須フィールド

他のKubernetes構成と同様、DaemonSetは`apiVersion`, `kind`, `metadata`フィールドが必要です。構成ファイルの利用についての一般的な情報は、[アプリケーションのデプロイ](/ja/docs/user-guide/deploying-applications/), [コンテナの構成](/ja/docs/tasks/), [kubectlを使ったオブジェクト管理](/ja/docs/concepts/overview/object-management-kubectl/overview/)のドキュメントを参照してください。

DaemonSetは.specセクションも必要です。

### Podテンプレート

`.spec.template`は`.spec`内に必要なフィールドのひとつである。

`.spec.template`は[Podテンプレート](/ja/docs/concepts/workloads/pods/pod-overview/#pod-templates)です。これは`apiVersion`や`kind`を持たないことを除いて、[Pod](/ja/docs/concepts/workloads/pods/pod/)と全く同じスキーマです。

Podに対して必要なフィールドに加え、DaemonSetのPodテンプレートは適切なラベルを指定しなければなりません ([Podセレクタ](#podセレクタ)を参照)。

DaemonSetのPodテンプレート`はAlways`である[`RestartPolicy`](/ja/docs/user-guide/pod-states)を持たねばならず、指定しない場合のデフォルトは`Always`です。

### Podセレクタ

`.spec.selector`フィールドはPodセレクタです。これは[Job](/ja/docs/concepts/jobs/run-to-completion-finite-workloads/)の`.spec.selector`と同様に動作します。

Kubernetes 1.8以降では、`.spec.template`のラベルとマッチするPodセレクタを指定しなければなりません。Podセレクタはもはや空のままの時にデフォルトが設定されることはありません。セレクタのデフォルト設定は`kubectl apply`と互換性がありません。また、一旦DaemonSetが作成されると、`.spec.selector`は変更できません。Podセレクタの変更は意図しないPodの孤立を招き、ユーザを混乱させます。

`.spec.selector`は2つのフィールドを含むオブジェクトです。

* `matchLabels`: [ReplicationController](/ja/docs/concepts/workloads/controllers/replicationcontroller/)の`.spec.selector`と同様に動作します
* `matchExpressions`: キーや値のリスト、キーと値に関連する演算子を指定することで、より洗練されたセレクタを構築できます。

2つとも指定された場合、AND結合された結果となります。

`.spec.selector`が指定されると、これは`.spec.template.metadata.labels`にマッチしなければなりません。これらにマッチしない構成ファイルはAPIによって拒絶されます。

また、このセレクタにマッチするラベルを持つPodを通常は作成すべきではありません。さもなければ、DaemonSetコントローラはこれらのPodがDaemonSetによって作成されたとみなしてしまいます。Kubernetesはユーザがこのような行動をとることを止めません。これを行いたくなるケースのひとつが、テスト目的で異なる値を持つPodを手動で作成することです。

### あるノードに限定してPodを実行する

もし`.spec.template.spec.nodeSelector`を指定すると、DaemonSetコントローラはその[ノードセレクタ](/ja/docs/concepts/configuration/assign-pod-node/)にマッチするノードにPodを作成します。同様に`.spec.template.spec.affinity`を指定すると、DaemonSetコントローラはその[ノードアフィニティ](/ja/docs/concepts/configuration/assign-pod-node/)にマッチするノードにPodを作成します。どちらも指定しなければ、DaemonSetコントローラはすべてのノードにPodを作成します。

## Daemon Podはどのようにスケジュールされるか

### DaemonSetコントローラによるスケジュール (デフォルト)

通常、Podを実行するマシンはKubernetesスケジューラによって選択されます。しかしながら、Daemonsetコントローラによって作成されたPodは既に選択されたマシンを持っています (Podが作成される時に`.spec.nodeName`が指定されるので、スケジューラによって無視される)。したがって、

 - ノードの[`unschedulable`](/ja/docs/admin/node/#manual-node-administration)フィールドはDaemonSetコントローラによって無視されます。
 - DaemonSetコントローラはスケジューラが起動していなくてもPodを作成でき、これはクラスタのブートを手助けできます。

### デフォルトスケジューラによるスケジュール

{{< feature-state state="alpha" for-kubernetes-version="1.11" >}}

DaemonSetはすべての的確なノードがPodのコピーを実行することを保証します。通常、Podを実行するノードはKubernetesスケジューラによって選択されます。しかし、DaemonSet Podは代わりにDaemonSetコントローラによって作成、スケジュールされます。これには以下のような問題があります。

 * 一貫性のないPodのふるまい: スケジュールされるのを待っている通常のPodは`Pending`状態で作成されるが、DaemonSet Podは`Pending`状態で作成されません。これはユーザを混乱させます。
 * [Podプリエンプション](/ja/docs/concepts/configuration/pod-priority-preemption/)はデフォルトスケジューラによって扱われます。プリエンプションが有効であっても、DaemonSetコントローラはPodの優先度やプリエンプションを考慮せずにスケジューリングを決定します。

`ScheduleDaemonSetPod`は、`.spec.nodeName`の代わりに`NodeAffinity`をDaemonSet Podに追加することで、DaemonSetコントローラではなくデフォルトスケジューラを使ったDaemonSetのスケジュールを可能にします。すると、デフォルトスケジューラがPodをターゲットホストに結びつけるために使われます。DaemonSet Podのノードアフィニティがすでに存在していれば、置き換えられます。DaemonSetコントローラは、DaemonSet Podを作成または編集する時にこれらの操作のみを実行し、DaemonSetの`.spec.template`は変更されません。

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchFields:
      - key: metadata.name
        operator: In
        values:
        - target-host-name
```

加えて、`node.kubernetes.io/unschedulable:NoSchedule`許容がDaemonSetに自動で追加されます。DaemonSetコントローラは、DaemonSet Podをスケジュールする時、`unschedulable`ノードを無視します。デフォルトスケジューラが同じ方法で振る舞い、DaemonSet Podを`unschedulable`ノードにスケジュールすることを保証するために`TaintNodesByCondition`を有効にしなければなりません。

この機能と`TaintNodesByCondition`が共に有効である時、DaemonSetがホストネットワークを使うのであれば、`node.kubernetes.io/network-unavailable:NoSchedule`許容も追加しなければなりません。

### 汚染と許容

Daemon Podは[汚染と許容](/ja/docs/concepts/configuration/taint-and-toleration)に従いますが、関連する機能に応じて以下の許容がDaemonSet Podに自動で追加されます。

| 許容キー                           | 効果     | アルファ機能                                               | バージョン | 説明                                                  |
| ---------------------------------------- | ---------- | ------------------------------------------------------------ | ------- | ------------------------------------------------------------ |
| `node.kubernetes.io/not-ready`           | NoExecute  | `TaintBasedEvictions`                                        | 1.8+    | `TaintBasedEvictions`が有効であれば、ネットワークパーティションのようなノードの問題がある際に追い出されない |
| `node.kubernetes.io/unreachable`         | NoExecute  | `TaintBasedEvictions`                                        | 1.8+    | `TaintBasedEvictions`が有効であれば、ネットワークパーティションのようなノードの問題がある際に追い出されない |
| `node.kubernetes.io/disk-pressure`       | NoSchedule | `TaintNodesByCondition`                                      | 1.8+    |                                                              |
| `node.kubernetes.io/memory-pressure`     | NoSchedule | `TaintNodesByCondition`                                      | 1.8+    |                                                              |
| `node.kubernetes.io/unschedulable`       | NoSchedule | `ScheduleDaemonSetPods`, `TaintNodesByCondition`             | 1.11+   | ` ScheduleDaemonSetPods`が有効であれば、DaemonsetPodがデフォルトスケジューラによる unschedulable属性を確実に許容するために、` TaintNodesByCondition`が必要です |
| `node.kubernetes.io/network-unavailable` | NoSchedule | `ScheduleDaemonSetPods`, `TaintNodesByCondition`, hostnework | 1.11+   | `ScheduleDaemonSetPods`が有効であれば、hostnetworkを使うDaemonSet Podがデフォルトスケジューラによるnetwork-unavailable属性を確実に許容するために、`TaintNodesByCondition`が必要である |
| `node.kubernetes.io/out-of-disk`         | NoSchedule | `ExperimentalCriticalPodAnnotation` (critical podのみ), `TaintNodesByCondition` | 1.8+    |                                                              |

## Daemon Podと通信する

DaemonSetのPodと通信する上でありえるパターンは以下です。

- **Push**: DaemonSet内のPodは状態データベースのような他のサービスへ更新を送信するよう構成される。クライアントを持たない。
- **NodeIPとKnown Port**: DaemonSet内のPodは`hostPort`を使うことができるので、PodはノードIP経由で到達可能である。クライアントは何らかの方法でノードIPのリストを知っており、慣例によりポートも知っている。
- **DNS**: 同じPodセレクタで[Headless Service](/ja/docs/concepts/services-networking/service/#headless-services)を作り、`endpoints`リソースまたはDNSから複数のAレコードを取得してDaemonSetを探す
- **Service**: 同じPodセレクタでServiceを作り、ランダムノードのデーモンに到達するためのそのServiceを使う。(特定のノードに到達するための方法はない。)

## DaemonSetの更新

ノードラベルが変更されると、DaemonSetは即座に新たにマッチするノードにPodを追加し、マッチしなくなったノードからPodを削除します。

DaemonSetが作成したPodは編集できます。しかしながら、Podはすべてのフィールドの更新を許可しているわけではありません。また、DaemonSetコントローラは次にノードが作成された場合に元のテンプレートを使います。

DaemonSetを削除することができます。`kubectl`で`--cascade=false`を指定すると、Podはノードに残ります。その後、異なるテンプレートの新しいDaemonSetを作ることができます。違うテンプレートの新しいDaemonSetは存在するPodをマッチするラベルを持つものと認識します。Podテンプレートと合致しないにもかかわらず、それらは編集または削除されません。

Kubernetes 1.6以降では、DaemonSetの[ローリングアップデートを実行](/ja/docs/tasks/manage-daemon/update-daemon-set/)できます。

## DaemonSetの代替

### Initスクリプト

デーモンプロセスをノードで(`init`や`upstartd`, `systemd`などを使って)直接起動することで実行することは確かに可能です。これは申し分なく良いことです。しかしながら、それらのプロセスをDaemonSet経由で起動することの利点がいくつかあります。

- アプリケーションごとに同じ方法でデーモンの監視とログ管理ができる
- デーモンとアプリケーションに対して同じ設定言語とツール(Podテンプレートや`kubectl`)が使える
- コンテナで実行しているデーモンのリソース制限の増大はデーモンとアプリケーションコンテナで分離される。しかし、これはデーモンをPodでなくコンテナで実行する(例えば直接Dockerを使う)ことにより達成される。

### Bare Pod

実行する特定のノードを指定して直接Podを作成することができます。しかし、DaemonSetは削除されたり、ノード障害やカーネルアップグレードのような破壊的なノードメンテナンスなど、何らかの理由で終了したPodを置き換えます。このため、個々のPodを作成するよりもDaemonSetを使うべきです。

### 静的Pod

kubeletが監視する特定のディレクトリにファイルを書くことでPodを作成することができます。これらは[静的Pod](/ja/docs/concepts/cluster-administration/static-pod/)と呼ばれます。DaemonSetと異なり、静的Podはkubectlや他のKubernetes APIクライアントでは管理できません。静的Podはapiserverに依存しないので、クラスタブートの場合に使うのが便利です。また、静的Podは将来非推奨になるかもしれません。

### Deployment

DaemonSetはPodを作成し、それらのPodは終了しないと想定されるプロセス(ウェブサーバやストレージサーバ)を持つという点でDeploymentに似ています。

Deploymentは、レプリカ数のスケールアップやダウン、ローリングアップデートが、正確にどのホストでPodを実行するのか制御することよりも重要な、フロントエンドのようなステートレスサービスで使います。

{{% /capture %}}
