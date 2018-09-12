---
title: Pod Lifecycle
content_template: templates/concept
weight: 30
---

{{% capture overview %}}

このページではPodのライフサイクルについて述べます。

{{% /capture %}}

{{% capture body %}}

## Podのフェーズ

Podの`status`フィールドは[PodStatus](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podstatus-v1-core)オブジェクトで、これはphaseフィールドを持ちます。

Podのフェーズはシンプルで、Podがどのライフサイクルにいるかについての高レベルな概要です。フェーズはコンテナまたはPodの監視の包括的なロールアップであることは意図されておらず、包括的なステートマシンであることも意図されていません。

Podフェーズ値の数字と意味は厳密に管理されています。ここで文書化されているもの以外、与えられた`phase`値を持つPodについて何も仮定すべきではありません。

`phase`の取り得る値を示します。

値 | 説明
:-----|:-----------
`Pending` | PodはKubernetesシステムに受理されたが、1つ以上のコンテナイメージが作成されていない。これにはネットワーク上でイメージをダウンロードする時間と同様、スケジュールされる前の時間も含まれる。
`Running` | Podがノードに割り当てられ、すべてのコンテナが作成されている。少なくとも1つのコンテナが起動しているか、中のプロセスが起動中か再起動中である。
`Succeeded` | Podのすべてのコンテナが正常に終了し、再起動されない。
`Failed` | Podのすべてのコンテナが終了し、少なくとも1つのコンテナが失敗して終了している。すなわち、コンテナは非ゼロ状態で終了したか、システムにより終了されたかである。
`Unknown` | 何らかの理由によりPodの状態が取得できない。通常はPodのホストと通信エラーが発生しているためである。

## Podの状態

Podは[PodCondition](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podcondition-v1-core)の配列を持つPodStatusを持ちます。PodCondition配列の各要素は6種類のフィールドを持ちます。

* `lastProbeTime`フィールドは最後にPodの状態を調べたタイムスタンプを提供する
* `lastTransitionTime`フィールドはPodがある状態から別の状態へ遷移したタイムスタンプを提供する
* `message`フィールドは遷移についての詳細を示す人間に可読なメッセージである
* `reason`フィールドは状態の最後の遷移についての一意で、一語のキャメルケースな理由である
* `status`フィールドは"`True`","`False`","`Unknown`"の値を取り得る文字列である
* `type`フィールドは以下の値を取り得る文字列である
  * `PodScheduled`: Podはノードにスケジュールされている
  * `Ready`: Podはリクエストを提供でき、すべてのマッチするServiceのロードバランシングプールに追加されるべきである
  * `Initialized`: すべての[initコンテナ](/ja/docs/concepts/workloads/pods/init-containers)が正常に開始している
  * `Unschedulable`: スケジューラは、リソース不足やその他の制約により、Podを今すぐにはスケジュールできない
  * `ContainersReady`: Podのすべてのコンテナは準備ができている

## コンテナの調査

[Probe](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#probe-v1-core)は[kubelet](/ja/docs/admin/kubelet/)によってコンテナで定期的に実行される診断です。診断を実行するために、kubeletはコンテナによって実装された[Handler](https://godoc.org/k8s.io/kubernetes/pkg/api/v1#Handler)を呼び出します。3種類のHandlerがあります。

* [ExecAction](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#execaction-v1-core): コンテナの中で指定されたコマンドを実行する。コマンドが終了コード0で終了すれば診断は成功とみなされる。
* [TCPSocketAction](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#tcpsocketaction-v1-core): コンテナIPアドレスの指定されたポートに対してTCPチェックを実行する。ポートが開いていれば診断は成功とみなされる。
* [HTTPGetAction](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#httpgetaction-v1-core): コンテナIPアドレスの指定されたポートとパスに対してHTTP Getリクエストを実行する。レスポンスが200以上400未満のステータスコードを持っていれば診断は成功とみなされる

各調査は以下の3つのうちの1つの結果になります。

* Success: コンテナは診断に成功した
* Failure: コンテナは診断に失敗した
* Unknown: 診断に失敗したため、取るべきアクションはない

kubeletは任意に実行中のコンテナで以下の2種類の調査を実行・反応できます。

* `livenessProbe`: コンテナが実行しているかどうかを示す。もしlivenessProbeに失敗すると、kubeletはコンテナをkillし、コンテナはその[再起動ポリシ](#再起動ポリシ)に従う。コンテナがlivenessProbeを提供しなければ、デフォルトの状態は`Success`となる。
* `readinessProbe`: コンテナがリクエストを処理する準備ができているかどうかを示す。もしreadinessProbeに失敗すると、エンドポイントコントローラはPodにマッチするすべてのServiceのエンドポイントからPodのIPアドレスを削除する。最初のディレイの前のデフォルト状態は`Failure`である。コンテナがreadinessProbeを提供しなければ、デフォルトの状態は`Success`となる。

### いつlivenessProbeやreadinessProbeを使うべきなのか

もしコンテナのプロセスが、問題に直面したり、調子が悪くなったりしたときに自身でクラッシュできるのであれば、livenessProbeを使う必要はありません。kubeletはPodの`restartPolicy`に応じて正しいアクションを自動で実行します。

調査に失敗した場合にコンテナをkillし再起動したいのであれば、livenessProbeを指定し、`restartPolicy`をAlwaysまたはOnFailureに指定します。

もし、調査が成功したときにのみトラフィックを送り始めたいのであれば、readinessProbeを指定します。この場合、readinessProbeはlivenessProbeと同じかもしれませんが、スペックでのreadinessProbeの存在は、Podがあらゆるトラフィックを受信することなく開始し、調査が成功し始めてからのみトラフィックを受信し始めるということを意味します。

コンテナが開始時に大きなデータや構成ファイルの読み込み、マイグレーションを行う必要があるのであれば、readinessProbeを指定します。

コンテナがメンテナンスのためにダウンできるようにしたいのであれば、livenessProbeとは異なるreadinessに特有のエンドポイントをチェックするreadinessProbeを指定できます。

Podが削除された時にリクエストを流せるようにしたいだけであれば、readinessProbeは必要ありません。削除時、PodはreadinessProbeが存在するかどうかにかかわらず、自動的にunreadyの状態に入ります。Pod中のコンテナが停止するまで、そのPodはundeadyの状態のままになります。

livenessとreadinessProbeをセットアップする方法についての詳細は、[LivenessとReadiness Probeの構成](/ja/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)を参照してください。

## Podとコンテナの状態

Podコンテナの状態についての詳細は、[PodStatus](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podstatus-v1-core)と[ContainerStatus](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#containerstatus-v1-core)を参照してください。現在の[ContainerState](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#containerstatus-v1-core)に依存するPodの状態として報告される情報に注意してください。

## Pod readiness gate

{{< feature-state for_k8s_version="v1.11" state="alpha" >}}

`PodStatus`への追加のフィードバックとシグナルの注入を有効にすることでPod readinessへの拡張を追加するために、Kubernetes 1.11では[Pod ready++](https://github.com/kubernetes/community/blob/master/keps/sig-network/0007-pod-ready%2B%2B.md)という機能を導入しました。Pod readinessを評価するための追加の条件を指定するために、`PodSpec`に新しいフィールドである`ReadinessGate`が使えます。KubernetesがPodの`status.conditions`フィールドにそのような条件を見つけられなければ、状態はデフォルトの"`False`"となります。例を以下に示します。

```yaml
Kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready  # this is a builtin PodCondition
      status: "True"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"   # an extra PodCondition
      status: "False"
      lastProbeTIme: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

新しいPodの状態はKubernetesの[ラベルキー形式](/ja/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)に従わなければなりません。`kubectl patch`コマンドはまだオブジェクト状態のパッチをサポートしないので、新しいPodの状態は[KubeClientライブラリ](/ja/docs/reference/using-api/client-libraries/)の1つを使って`PATCH`アクションを通じて注入する必要があります。

新しいPodの状態の導入で、Podは以下が両方とも真である時 **のみ** 、準備完了であると評価される。

* Pod内のすべてのコンテナが準備完了である
* ReadinessGatesで指定されたすべての状態が"True"である

この変更をPod readiness評価で手助けするため、新しいPod状態 `ContainersReady`が古いPodの`Ready`状態をキャプチャするために導入されます。

Pod Ready++はアルファ機能であるため、`PodReadinessGates` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) をTrueに設定することで、明示的に有効にしなければなりません。

## 再起動ポリシ

PodSpecはAlways, OnFailure, Neverの値を取り得る`restartPolicy`フィールドを持ちます。デフォルトの値はAlwaysである。`restartPolicy`はPod内のすべてのコンテナに適用されます。`restartPolicy`は同じノードのkubeletによりコンテナの再起動時にのみ参照されます。kubeletによって再起動される終了したコンテナは5分を限度とする指数的なバックオフディレイ(10s, 20s, 40s,...) 後に再起動され、成功した実行の10分後にリセットされます。[Podのドキュメント](/ja/docs/user-guide/pods/#durability-of-pods-or-lack-thereof)で議論してように、一度ノードに結合されると、Podは決してほかのノードに再結合されることはありません。

## Podの生存期間

通常、Podは誰かが削除するまで消えません。誰かというのは人間もしくはコントローラです。このルールの唯一の例外は、(マスタの`terminated-pod-gc-threshold`で決められる)ある期間以上SucceededもしくはFailedの`phase`であるPodが期限切れとなり自動的に削除されることでです。

3種類のコントローラが利用可能です。

- バッチ計算のような、終了すると期待されるPodに対しては[Job](/ja/docs/concepts/jobs/run-to-completion-finite-workloads/)を使う。Jobは`restartPolicy`がOnFailureまたはNeverのPodに対してのみ適している。
- Webサーバのような、終了すると期待されないPodに対しては[ReplicationController](/ja/docs/concepts/workloads/controllers/replicationcontroller/), [ReplicaSet](/ja/docs/concepts/workloads/controllers/replicaset/), [Deployment](/ja/docs/concepts/workloads/controllers/deployment/)を使う。ReplicationControllerは`restartPolicy`がAlwaysであるPodに対してのみ適している。
- マシン固有のシステムサービスを提供するために、マシンごとに1つ実行する必要のあるPodに対しては[DaemonSet](/docs/concepts/workloads/controllers/daemonset/)を使う

3種類のコントローラにはすべてPodTemplateが含まれます。Podを直接作成するよりも、適したコントローラを作成し、Podを作成させることを推奨します。単体のPodはマシン障害への回復力がありませんが、コントローラはそうでないためです。

ノードが死ぬかクラスタからの接続が断たれると、Kubernetesは失われたノードのすべてのPodの`phase`をFailedに設定するポリシを適用します。

## 例

### 高度な生存調査の例

生存調査はkubeletによって実行されるので、すべてのリクエストはkubeletのネットワーク名前空間で作成されます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - args:
    - /server
    image: k8s.gcr.io/liveness
    livenessProbe:
      httpGet:
        # "host"が定義されると、"PodIP"が使われます
        # host: my-host
        # "scheme"が定義されないと、"HTTP"スキームが使われます。"HTTP"と"HTTPS"のみが許可されます
        # scheme: HTTPS
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      timeoutSeconds: 1
    name: liveness
```

### 例の状態

   * Podが実行中でコンテナを1つ持つ。コンテナは正常に終了する。
     * 完了イベントがログに記録される。
     * `restartPolicy`が
       * Alwaysであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
       * OnFailureであれば、Podの`phase`はSucceededになる。
       * Neverであれば、Podの`phase`はSucceededになる。

   * Podが実行中でコンテナを1つ持つ。コンテナが異常終了する。
     * 失敗イベントがログに記録される。
     * `restartPolicy`が
       * Alwaysであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
       * OnFailureであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
       * Neverであれば、Podの`phase`はFailedになる

   * Podが実行中で2つのコンテナを持つ。1つのコンテナが異常終了する。
     * 失敗イベントがログに記録される
     * `restartPolocy`が
       * Alwaysであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
       * OnFailureであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
       * Neverであれば、コンテナは再起動されず、Podの`phase`はRunningのままである。
     * コンテナ1 が実行されておらず、コンテナ2が終了する
       * 失敗イベントがログに記録される
       * `restartPolicy`が
         * Alwaysであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
         * OnFailureであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
         * Neverであれば、Podの`phase`はFailedになる

   * Podが実行中でコンテナを1つ持つ。コンテナはメモリ不足になる。
     * コンテナは失敗で終了する。
     * OOMイベントがログに記録される
     * `restartPolicy`が
       * Alwaysであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
       * OnFailureであれば、コンテナが再起動され、Podの`phase`はRunningのままである。
       * Neverであれば、失敗イベントがログに記録され、Podの`phase`はFailedになる。

   * Podが実行中で、ディスクが死ぬ。
     * すべてのコンテナがkillされる。
     * 適切なイベントがログに記録される。
     * Podの`phase`はFailedになる。
     * コントローラ下で実行していれば、Podは他の場所に再作成される。

   * Podが実行中で、そのノードが分断される。
     * ノードコントローラはタイムアウトまで待つ。
     * ノードコントローラはPodの`phase`をFailedに設定する。
     * コントローラ下で実行していれば、Podは他の場所に再作成される。

{{% /capture %}}

{{% capture whatsnext %}}

* [コンテナライフサイクルイベントにハンドラを付加する](/ja/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)でハンズオンを経験します

* [liveness と readiness probeを構成する](/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)でハンズオンを経験します

* [コンテナライフサイクルフック](/docs/concepts/containers/container-lifecycle-hooks/)について更に学びます

{{% /capture %}}
