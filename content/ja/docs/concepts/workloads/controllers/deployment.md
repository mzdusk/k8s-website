---
title: Deployments
feature:
  title: 自動化されたロールアウトとロールバック
  description: >
    Kubernetesは、同時にすべてのインスタンスがダウンしないようにアプリケーションの状態を監視しながら、アプリケーションや構成への変化を漸進的にロールアウトします。異常があれば、Kubernetesは変更のロールバックを行います。成長しているデプロイソリューションのエコシステムを活用しましょう。

content_template: templates/concept
weight: 30
---

{{% capture overview %}}

_Deployment_ コントローラは[Pods](/docs/concepts/workloads/pods/pod/)と[ReplicaSets](/docs/concepts/workloads/controllers/replicaset/)に対する宣言的な更新を提供します。

Deploymentオブジェクトに _求める状態_ を記述し、Deploymentコントローラは現在の状態を求める状態に制御された速度で変更します。新しいReplicaSetを作成するか、既存のDeploymentを削除して新しいリソースに適合するようDeploymentを定義できます。

{{< note >}}
**メモ:** Deploymentが保持するReplicaSetを管理すべきではありません。そのようなユースケースはすべてDeploymentの操作で対応すべきです。以下で取り扱わないユースケースがあれば、メインのKubernetesリポジトリにIssueをオープンすることを検討してください。
{{< /note >}}

{{% /capture %}}

{{% capture body %}}

## ユースケース {#use-case}

以下がDeploymentの典型的なユースケースです。

* [ReplicaSetをロールアウトするためにDeploymentを作成する](#creating-a-deployment)。ReplicaSetは裏でPodを作成します。成功したかどうかを見るために、ロールアウトのステータスをチェックしてください。
* DeploymentのPodTemplateSpecを更新することで、[Podの新しい状態を宣言する](#updating-a-deployment)。新しいReplicaSetが作られ、Deploymentは古いReplicaSetから新しいものへのPodの移動を管理します。各新しいReplicaSetはDeploymentのリビジョンを更新します。
* Deploymentの現在の状態が不安定になった場合、[以前のDeploymentリビジョンにロールバックする](#rolling-back-a-deployment)。各ロールバックはDeploymentのリビジョンを更新します。
* [Deploymentをスケールアップする](#scaling-a-deployment)。
* PodTemplateSpecへの複数の修正を適用するために[Deploymentを停止し](#pausing-and-resuming-a-deployment)、新しいロールアウトを開始するために再開する。
* ロールアウトが失敗したことを示すものとして、[Deploymentのステータスを使う](#deployment-status)
* 必要のない[古いReplicaSetをクリーンアップする](#clean-up-policy)


## Deploymentを作成する {#creating-a-deployment}

以下はDeploymentの例です。3つの`nginx` Podを起動するReplicaSetを作成します。

{{< codenew file="controllers/nginx-deployment.yaml" >}}

この例では次のようなことをします。

* `.metadata.name`フィールドで指定された、`nginx-deployment`という名前のDeploymentが作成される
* `replicas`フィールドで指定されたように、このDeploymentは3レプリカのPodを作成する
* `selector`フィールドは、Deploymentが管理対象のPodをどのように見つけるのかを定義する。この場合、Podテンプレートで定義されたラベル (`app: nginx`) を単純に選択する。Podテンプレート自身がルールを満たすのであれば、より複雑な選択ルールも利用可能である。

  {{< note >}}
  **メモ:** `matchLabels`は {key,value} ペアのマップです。`matchLabels`マップでの単一の {key,value} は、`matchExpressions`でキーフィールドに"key"、演算子に"In"、値の配列に"value"のみを指定する場合と等価です。必要条件はANDで結合されます。
  {{< /note >}}

* `template`フィールドには以下の下位フィールドを含む
  * Podは`labels`フィールドを使って`app: nginx`とラベル付けされる
  * Podテンプレートのスペック、あるいは`.template.spec`は、PodがVersion 1.15.4の`nginx` [Docker Hub](https://hub.docker.com/)イメージを実行する`nginx`という1つのコンテナを実行するよう指示する
  * `name`フィールドを使って、`nginx`という名前のコンテナを1つ作成する
  * Version `1.15.4`の`nginx`イメージを実行する
  * コンテナがトラフィックの送受信が可能な`80`番ポートを開ける

このDeploymentを作成するために、以下のコマンドを実行します。

```shell
kubectl create -f  https://k8s.io/examples/controllers/nginx-deployment.yaml
```

{{< note >}}
**メモ:** `kubernetes.io/change-cause`というリソースアノテーションで実行されるコマンドを書くために、`--record`フラグを指定できます。これは、例えば、各Deploymentのリビジョンで実行されたコマンドを見るなど、将来のイントロスペクションに便利です。
{{< /note >}}

次に、`kubectl get deployments`を実行します。出力は以下のようになります。

```shell
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```

クラスタ内のDeploymentを調べるために、以下のフィールドが表示されます。

* `NAME`はクラスタ内のDeploymentの名前を列挙する
* `DESIRED`は、Deploymentを作成する時に定義した、希望されるアプリケーションの _レプリカ_ の数を表示する。これが _希望される状態_ である
* `CURRENT`は現在実行中のレプリカ数を表示する
* `UP-TO-DATE`は希望される状態を達成するまえに更新されたレプリカの数を表示する
* `AVAILABLE`はユーザが利用可能なアプリケーションのレプリカ数を表示する
* `AGE`はアプリケーションの実行時間を表示する

各フィールドの値がDeploymentスペックの値とどのように対応するのかについて示します。

* 希望されるレプリカ数の3は、`.spec.replicas`フィールドに対応する
* 現在のレプリカ数の0は、`.status.replicas`フィールドに対応する
* 更新されたレプリカ数の0は、`.status.updatedReplicas`フィールドに対応する
* 利用可能なレプリカ数の0は、`.status.availableReplicas`フィールドに対応する

Deploymentのロールアウトステータスを見るために、`kubectl rollout status deployment/nginx-deployment`を実行します。このコマンドは以下のような出力を返します。

```shell
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```

数秒後に再び`kubectl get deployments`を実行します。

```shell
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           18s
```

Deploymentは3つのレプリカすべてを作成し、そのすべてのレプリカが最新 (最新のPodテンプレートを含む) で利用可能 (少なくともDeploymentの`.spec.minReadySeconds`の時間、PodステータスがReadyになっている) な状態です。

Deploymentによって作成されたReplicaSet (`rs`) を見るために、`kubectl get rs`を実行します。

```shell
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-2035384211   3         3         3       18s
```

ReplicaSetの名前は常に`[DEPLOYMENT-NAME]-[POD-TEMPLATE-HASH-VALUE]`の形式で表されます。ハッシュ値はDeploymentが作成された時に自動で生成されます。

各Podに対して自動的に生成されたラベルを見るために`kubectl get pods --show-labels`を実行します。以下の出力が返ります。

```shell
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-2035384211-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
nginx-deployment-2035384211-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
nginx-deployment-2035384211-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
```

作成されたReplicaSetは常に3つの`nginx` Podが起動することを保証します。

{{< note >}}
**メモ:** Deploymentでは適切なセレクタとPodテンプレートラベルを指定しなければなりません (この場合は`app: nginx`)。他のコントローラ (他のDeploymentやStatefulSetを含む) とラベルやセレクタが重複してはいけません。Kubernetesが重複を止めることはできず、複数のコントローラが重複するセレクタを持てば、これらのコントローラは衝突を起こし、予期しない動作をする可能性があります。
{{< /note >}}

### pod-template-hashラベル {#pod-template-hash-label}

{{< note >}}
**メモ:** このラベルを変えてはいけません。
{{< /note >}}

`pod-template-hash`ラベルは、Deploymentが作成または選択したすべてのReplicaSetに追加されます。

このラベルは、Deploymentの子のReplicaSetが重複しないことを保証します。これはReplicaSetの`PodTemplate`をハッシュ化し、そのハッシュ値をReplicaSetセレクタやPodテンプレートラベル、ReplicaSetが持つ既存のPodに追加されるラベル値として使うことで生成されます。

## Deploymentの更新 {#updating-a-deployment}

{{< note >}}
**メモ:** Deploymentのロールアウトは、DeploymentのPodテンプレート (すなわち `.spec.template`) が変更される (例えば、ラベルやテンプレートのコンテナイメージが更新される) ことで起動します。スケーリングのような、その他の更新ではロールアウトは起動しません。
{{< /note >}}

Podが使うnginxを`nginx:1.7.9`から`nginx:1.9.1`に更新したいとします。

```shell
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record
deployment.apps/nginx-deployment image updated
```

または、Deploymentを`edit`し、`.spec.template.spec.containers[0].image`を`nginx:1.7.9`から`nginx:1.9.1`に変更します。

```shell
$ kubectl edit deployment/nginx-deployment
deployment.apps/nginx-deployment edited
```

ロールアウトステータスを見るために以下を実行します。

```shell
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```

ロールアウトが成功した後、Deploymentを`get`したくなるかもしれません。

```shell
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           36s
```

更新レプリカの数はDeploymentがレプリカを最新の構成に更新したことを示します。現在のレプリカは、このDeploymentが管理するトータルのレプリカを示し、利用可能レプリカは現在利用可能なレプリカ数を示します。

Deploymentが新しいReplicaSetを作成し、それを3レプリカにスケールアップすることによってPodを更新したことを見るために、`kubectl get rs`を実行します。

```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       6s
nginx-deployment-2035384211   0         0         0       36s
```

`get pods`は新しいPodのみを表示するでしょう。

```shell
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1564180365-khku8   1/1       Running   0          14s
nginx-deployment-1564180365-nacti   1/1       Running   0          14s
nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
```

次にこれらのPodを更新したいのであれば、DeploymentのPodテンプレートを再び更新するだけです。

Deploymentは、更新時にある数のPodだけがダウンすることを保証します。デフォルトでは、利用できないPodが希望する数の25%未満になることを保証します。

Deploymentは、希望するPod数を超えて作成されるPodの数をある数だけにすることも保証します。デフォルトでは、増加数が最大で希望する数の25%になることを保証します。

例えば、上のDeploymentをよく見ると、最初に新しいPodを作成して、その後にいくつか古いPodを削除して、新しいものを削除していることがわかります。これは、十分な数の新しいPodが立ち上がるまで古いPodを削除せず、十分な数の古いPodが削除されるまで新しいPodを作成しません。これは、利用可能なPodが2以上で、Podの総数が4以下になるようにします。

```shell
$ kubectl describe deployments
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 30 Nov 2017 10:56:25 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=2
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.9.1
    Port:         80/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-1564180365 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-2035384211 to 3
  Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 1
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 2
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 2
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 1
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 3
  Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 0
```

これを見ると、最初にDeploymentを作成した時に、ReplicaSet (nginx-deployment-2035384211) が作られ、それがレプリカ数3に直接スケールアップされている様子がわかります。Deploymentを更新した場合、新しいReplicaSet (nginx-deployment-1564180365) が作られ、それが1にスケールアップし、その後古いReplicaSetが2にスケールダウンされています。したがって、少なくとも2つのPodが利用可能で、多くても4つのPodが同時に作られます。その後、新しいReplicaSetと古いReplicaSetでスケールアップとスケールダウンがくり返されます。最後に、新しいReplicaSetの利用可能レプリカ数が3になり、古いReplicaSetは0にスケールダウンされます。

### ロールオーバ (別名、実行中の複数アップデート)

毎回、新しいDeploymentオブジェクトはDeploymentコントローラに監視され、既存のReplicaSetがなければ、希望するPodを立ち上げるために作成されます。`.spec.selector`がラベルにマッチするが、`.spec.template`がテンプレートにマッチしないPodをコントロールしている既存のReplicaSetはスケールダウンされます。最終的に、新しいReplicaSetは`.spec.replicas`にスケールされ、すべての古いReplicaSetは0にスケールされます。

ロールアウトが進行中にDeploymentを更新すると、Deploymentは更新のために新しいReplicaSetを作成してスケールアップを開始し、以前にスケールアップされたReplicaSetをロールオーバします -- 古いReplicaSetのリストに追加され、スケールダウンが始まります。

例えば、5レプリカの`nginx:1.7.9`を作成するためにDeploymentを作成したものの、3レプリカの`nginx:1.7.9`が作成された時点で、5レプリカの`nginx:1.9.1`を作成するためにDeploymentを更新したとします。この場合、Deploymentは即座に、作成済みの3つの`nginx:1.7.9` Podを削除し始め、`nginx:1.9.1`のPodを作成しはじめます。`nginx:1.7.9`のレプリカ数が5になるまで待ちはしません。

### ラベルセレクタの更新 {#label-selector-updates}

一般にラベルセレクタの更新は避けるべきで、あらかじめ計画しておくようお勧めします。ラベルセレクタの変更が必要であれば、最大限の注意をもって実行し、考えられる結果のすべてを確実に把握してください。

{{< note >}}
**メモ:** APIバージョン`apps/v1`では、Deploymentのラベルセレクタは作成後にイミュータブルです。
{{< /note >}}

* セレクタの追加は、DeploymentスペックのPodテンプレートラベルを新しいラベルに更新する際に必要で、そうしないと検証エラーが返ってきます。この変更は重複のないもので、新しいセレクタが古いセレクタによって作成されたReplicaSetとPodを選択しないことを意味していて、その結果、古いReplicaSetはすべて孤立し、新しいReplicaSetが作成されます。
* セレクタの更新 -- すなわち、既存のセレクタキーの値の変更 -- は追加の時と同じ動作になります。
* セレクタの削除 -- すなわち、Deploymentセレクタからの既存キーの削除 -- はPodテンプレートラベルの変更を必要としません。存在しないReplicaSetは孤立し、新しいReplicaSetは作成されませんが、削除されたラベルは既存のPodとReplicaSetに残り続けます。

## Deploymentのロールバック {#rolling-back-a-deployment}

時々Deploymentをロールバックしたくなることがあるかもしれません。例えば、Deploymentがクラッシュループなどで不安定になった場合などです。デフォルトでは、Deploymentのロールアウト履歴はすべてシステムに保持されるので、いつでもロールバックできます。

{{< note >}}
**メモ:** DeploymentのリビジョンはDeploymentのロールアウトが開始された時に作成されます。つまり、新しいリビジョンは、ラベルやコンテナイメージの更新など、DeploymentのPodテンプレート (`.spec.template`) が変更された場合にのみ作成されるということです。Deploymentのスケーリングなど、その他の更新ではDeploymentリビジョンが作成されませんので、同時に手動や自動のスケーリングをすることが容易です。すなわち、以前のリビジョンにロールバックする場合、DeploymentのPodテンプレートの部分のみがロールバックされます。
{{< /note >}}

Deploymentを更新する時にタイポして、イメージ名を`nginx:1.9.1`ではなく`nginx:1.91`にしてしまったとします。

```shell
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment.apps/nginx-deployment image updated
```

ロールアウトは失敗します。

```shell
$ kubectl rollout status deployments nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
```

Ctrl-Cを押してロールアウトステータスの監視を止めます。失敗したロールアウトの詳細は、[こちらを参照してください](#deployment-status)。

古いレプリカ (nginx-deployment-1564180365とnginx-deployment-2035384211) の数と新しいレプリカ (nginx-deployment-3066724191) の数が両方とも2になっていることがわかります。

```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   2         2         2       25s
nginx-deployment-2035384211   0         0         0       36s
nginx-deployment-3066724191   2         2         0       6s
```

作成されたPodを見ると、新しいReplicaSetによって作成された2つのPodがImage Pull Loopで失敗していることがわかります。

```shell
$ kubectl get pods
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1       Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
nginx-deployment-3066724191-eocby   0/1       ImagePullBackOff   0          6s
```

{{< note >}}
**メモ:** Deploymentコントローラは悪いロールアウトを自動で止め、新しいReplicaSetのスケールアップを止めます。これは指定されたrollingUpdateパラメータ (特に`maxUnavailable`) によります。デフォルトでは、Kubernetesはこの値を1に、`.spec.replicas`を1に設定するので、これらのパラメータを変更しなかった場合、Deploymentはデフォルトで100%利用不可になります! これは将来のバージョンで修正される予定です。
{{< /note >}}

```shell
$ kubectl describe deployment
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       2 updated | 3 total | 2 available | 2 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:     nginx-deployment-1564180365 (2/2 replicas created)
NewReplicaSet:      nginx-deployment-3066724191 (2/2 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  1m        1m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-1564180365 to 2
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 2
```

これを修正するために、安定していた以前のリビジョンにDeploymentをロールバックする必要があります。

### Deploymentのロールアウト履歴のチェック {#checking-rollout-history-of-a-deployment}

まず、このDeploymentの現在のリビジョンをチェックします。

```shell
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create --filename=https://k8s.io/examples/controllers/nginx-deployment.yaml --record=true
2           kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record=true
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91 --record=true
```

`CHANGE-CAUSE`は、そのリビジョンが作成された時の`kubernetes.io/change-cause`というDeploymentアノテーションからコピーされます。以下のようにして`CHANGE-CAUSE`メッセージを指定できます。

* `kubectl annotate deploy nginx-deployment kubernetes.io/change-cause="image update to 1.9.1`を実行してDeploymentにアノテーションを付加する
* リソースを変更させる`kubectl`コマンドを保存するために`--record`フラグを追加する
* リソースのマニフェストを手動で編集する

各リビジョンの詳細を見るために、以下を実行します。

```shell
$ kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Annotations:  kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record=true
  Containers:
   nginx:
    Image:      nginx:1.9.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```

### 以前のリビジョンへのロールバック {#rolling-back-to-a-previous-revision}

以下のコマンドで、現在のロールアウトを以前のリビジョンにロールバックします。

```shell
$ kubectl rollout undo deployment/nginx-deployment
deployment.apps/nginx-deployment
```

別の方法として、`--to-revision`を指定することで、指定したリビジョンにロールバックすることもできます。

```shell
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment.apps/nginx-deployment
```

ロールアウト関連のコマンドについては、[`kubectl rollout`](/docs/reference/generated/kubectl/kubectl-commands#rollout)を参照してください。

これでDeploymentは以前の安定していたリビジョンにロールバックされました。リビジョン2へロールバックされたという`DeploymentRollback`イベントがDeploymentコントローラによって生成されます。

```shell
$ kubectl get deployment nginx-deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           30m

$ kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 02 Sep 2018 18:17:55 -0500
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=4
                        kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record=true
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.9.1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-c4747d96c (3/3 replicas created)
Events:
  Type    Reason              Age   From                   Message
  ----    ------              ----  ----                   -------
  Normal  ScalingReplicaSet   12m   deployment-controller  Scaled up replica set nginx-deployment-75675f5897 to 3
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 1
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 2
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 2
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 1
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 3
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 0
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-595696685f to 1
  Normal  DeploymentRollback  15s   deployment-controller  Rolled back deployment "nginx-deployment" to revision 2
  Normal  ScalingReplicaSet   15s   deployment-controller  Scaled down replica set nginx-deployment-595696685f to 0
```

## Deploymentのスケーリング {#scaling-a-deployment}

以下のコマンドを使うことで、Deploymentをスケールできます。

```shell
$ kubectl scale deployment nginx-deployment --replicas=10
deployment.apps/nginx-deployment scaled
```

[水平Podオートスケーリング](/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)がクラスタで有効になっていると、Deploymentに対してオートスケーラをセットアップし、PodのCPU使用量に基いてPodの最小値と最大値を選択することができます。

```shell
$ kubectl autoscale deployment nginx-deployment --min=10 --max=15 --cpu-percent=80
deployment.apps/nginx-deployment scaled
```

### 比例スケーリング {#proportional-scaling}

RollingUpdate Deploymentは複数バージョンのアプリケーションの同時実行をサポートします。ユーザやオートスケーラがロールアウト中 (進行中か停止中のどちらか) のRollingUpdate Deploymentをスケールする場合、Deploymentコントローラは、リスクを軽減するため、既存のアクティブなReplicaSet (Podを持つReplicaSet) にレプリカを追加してバランスを取ります。これは *比例スケーリング* と呼ばれます。

例えば、[maxSurge](#max-surge)=3、[maxUnavailable](#max-unavailable)=2でレプリカ数が10のDeploymentを実行します。

```shell
$ kubectl get deploy
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     10        10        10           10          50s
```

クラスタ内で解決できない問題が発生するイメージに更新します。

```shell
$ kubectl set image deploy/nginx-deployment nginx=nginx:sometag
deployment.apps/nginx-deployment image updated
```

イメージの更新は、nginx-deployment-1989198191というReplicaSetの新しいロールアウトを開始しますが、上で言及した`maxUnavailable`条件のためにブロックされます。

```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   5         5         0         9s
nginx-deployment-618515232    8         8         8         1m
```

すると、新しいスケーリングリクエストがやってきます。オートスケーラはDeploymentのレプリカ数を15に増やします。Deploymentコントローラは、新しい5つのレプリカを追加する場所を決める必要があります。比例スケーリングを使わなければ、5つはすべて新しいReplicaSetに追加されます。比例スケーリングでは、追加のレプリカをReplicaSet全体に広げます。大きい方の比率が多くのレプリカを持つReplicaSetに割り当てられ、小さい方の比率が少ないレプリカを持つReplicaSetに割り当てられます。余りは多くのレプリカを持つReplicaSetに加えられます。0レプリカのReplicaSetはスケールアップしません。

上の例では、3つのレプリカが古いReplicaSetに加えられ、2つのレプリカが新しいReplicaSetに加えられます。新しいレプリカは正常になると考えられるので、ロールアウトプロセスは徐々にすべてのレプリカを新しいReplicaSetに移動させます。

```shell
$ kubectl get deploy
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   7         7         0         7m
nginx-deployment-618515232    11        11        11        7m
```

## Deploymentの停止と再開 {#pausing-and-resuming-a-deployment}

1つまたは複数の更新が始まえう前にDeploymentを停止させ、その後再開させることができます。これは停止と再開の間に不要なロールアウトを開始させることなく、複数の修正を適用できるようにします。

例えば、作られたばかりのDeploymentがあるとします。

```shell
$ kubectl get deploy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     3         3         3            3           1m
$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         1m
```

以下のコマンドを実行して、停止させます。

```shell
$ kubectl rollout pause deployment/nginx-deployment
deployment.apps/nginx-deployment paused
```

新しいロールアウトが始まっていないことを確認します。

```shell
$ kubectl rollout history deploy/nginx-deployment
deployments "nginx"
REVISION  CHANGE-CAUSE
1   <none>

$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         2m
```

必要なだけの更新を行うことができます。例えば、使用するリソースを更新します。

```shell
$ kubectl set resources deployment nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
deployment.apps/nginx-deployment resource requirements updated
```

停止する前のDeploymentの最初の状態でその機能を継続しますが、Deploymentへの新しい更新はDeploymentが停止している間は効果がありません。

その後、Deploymentを再開し、新しい更新で立ち上がるReplicaSetを観察します。

```shell
$ kubectl rollout resume deploy/nginx-deployment
deployment.apps/nginx-deployment resumed
$ kubectl get rs -w
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   2         2         2         2m
nginx-3926361531   2         2         0         6s
nginx-3926361531   2         2         1         18s
nginx-2142116321   1         2         2         2m
nginx-2142116321   1         2         2         2m
nginx-3926361531   3         2         1         18s
nginx-3926361531   3         2         1         18s
nginx-2142116321   1         1         1         2m
nginx-3926361531   3         3         1         18s
nginx-3926361531   3         3         2         19s
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         20s
^C
$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         28s
```

{{< note >}}
**メモ:** 停止しているDeploymentのロールバックは、再開するまでできません。
{{< /note >}}

## Deploymentのステータス {#deployment-statue}

Deploymentはそのライフサイクルで様々な状態をとります。新しいReplicaSetをロールアウトしている間は[progressing](#progressing-deployment)となり、その後、[complete](#complete-deployment)や[fail to progress](#failed-deployment)となります。

### 進行中のDeployment {#progressing-deployment}

Kubernetesは、以下のタスクが実行されている場合、Deploymentを _progressing_ とします。

* Deploymentが新しいReplicaSetを作成する
* Deploymentが最新のReplicaSetをスケールアップしている
* Deploymentが古いReplicaSetをスケールダウンしている
* 新しいPodがReadyもしくはAvailableになる (少なくとも[MinReadySeconds](#min-ready-seconds)の間はReadyです)

Deploymentの進行状況は`kubectl rollout status`を使って監視できます。

### 完了したDeployment {#complete-deployment}

Kubernetesは、以下の状態になった場合、Deploymentを _complete_ とします。

* Deploymentに関連するすべてのレプリカが、指定された最新バージョンに更新される。すなわち、リクエストされたあらゆる更新が完了する
* Deploymentに関連するすべてのレプリカがAvailableになる
* Deploymentの古いレプリカが実行していない

Deploymentが完了しているかどうかは、`kubectl rollout status`を使ってチェックできます。ロールアウトが正常に完了していれば、`kubectl rollout status`は終了コード0を返します。

```shell
$ kubectl rollout status deploy/nginx-deployment
Waiting for rollout to finish: 2 of 3 updated replicas are available...
deployment.apps/nginx-deployment successfully rolled out
$ echo $?
0
```

### 失敗したDeployment {#failed-deployment}

Deploymentは新しいReplicaSetのデプロイしようとして失敗する可能性があります。これは以下のような理由で起こることがあります。

* 不十分なクオータ
* Readiness Probeの失敗
* イメージPullエラー
* 不十分な権限
* 制限に到達
* アプリケーションの構成ミス

この状況を検知する方法の1つが、Deploymentスペックにデッドラインパラメータ ([`.spec.progressDeadlineSeconds`](#progress-deadline-seconds)) を指定することです。`.spec.progressDeadlineSeconds`は、Deploymentの進行が失敗したと判断する前にDeploymentコントローラが待つ秒数を表します。

以下の`kubectl`コマンドは、10分後にDeploymentの進行が止まっていることをコントローラが報告するよう、`progressDeadlineSeconds`のスペックを設定します。

```shell
$ kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
deployment.apps/nginx-deployment patched
```

デッドラインを過ぎると、Deploymentコントローラは以下の属性を持つDeploymentConditionをDeploymentの`.status.condition`に追加します。

* Type=Progressing
* Status=False
* Reason=ProgressDeadlineExceeded

ステータスの状態についての詳細は[Kubernetes API conventions](https://git.k8s.io/community/contributors/devel/api-conventions.md#typical-status-properties)を参照してください。

{{< note >}}
**メモ:** Kubernetesは失敗したDeploymentに対して、`Reason=ProgressDeadlineExceeded`のステータス状態を報告する以外の動作は行いません。高水準のオーケストレータがこれを利用して、例えばDeploymentを以前のバージョンにロールバックするなど、適切に動作します。
{{< /note >}}

{{< note >}}
**メモ:** Deploymentを停止していると、Kubernetesは指定されたデッドラインに対して進行をチェックしません。ロールアウトの途中でも安全にDeploymentを停止し、デッドラインに達したと判断されることなく再開できます。
{{< /note >}}

設定したタイムアウトが短かかったり、一時的であると扱われるエラーによって、Deploymentが一時的なエラーを発生させるかもしれません。例えば、不十分なクオータがあるとします。Deploymentを記述すれば、以下のようなセクションがあることに気付くでしょう。

```shell
$ kubectl describe deployment nginx-deployment
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```

`kubectl get deployment nginx-deployment -o yaml`を実行すると、Deploymentステータスはこのようになります。

```
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
```

そのうち、Deploymentプロセスのデッドラインに達し、Kubernetesは進行中の状況のステータスと理由を更新します。

```
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     False   ProgressDeadlineExceeded
  ReplicaFailure  True    FailedCreate
```

不十分なクオータの問題には、Deploymentのスケールダウン、稼動しているその他のコントローラのスケールダウン、クオータの増強で対処できます。クオータの状況とDeploymentコントローラが満たされれば、Deploymentのロールアウトが完了し、Deploymentのステータスは正常な状態に更新されます (`Statue=True`と`Reason=NewReplicaSetAvailable`)。

```
Conditions:
  Type          Status  Reason
  ----          ------  ------
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
```

`Status=True`の`Type=Available`はDeploymentが最低限利用可能であるという意味です。最低限利用可能であることの条件は、Deployment戦略で指定されたパラメータで決まります。`Status=True`の`Type=Processing`は、Deploymentがロールアウト中であるか、新しいレプリカの最低必要数が利用可能であるかのどちらかです (詳細は状態の理由を見てください。この場合、`Reason=NewReplicaSetAvailable`はDeploymentが完了したことを意味します)。

`kubectl rollout status`を使って、Deploymentが失敗したかどうかをチェックできます。`kubectl rollout status`はDeploymentが進行デッドラインに達すると非ゼロの終了コードを返します。

```shell
$ kubectl rollout status deploy/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
error: deployment "nginx" exceeded its progress deadline
$ echo $?
1
```

### 失敗したDeploymentの操作 {#operating-on-a-failed-deployment}

完了したDeploymentに適用されたすべてのアクションは、失敗したDeploymentにも適用されます。スケールダウン/アップや、以前のリビジョンへのロールバック、DeploymentのPodテンプレートへ複数の調整を加える必要がある場合の停止でさえ適用できます。

## クリーンアップポリシ {#clean-up-policy}

古いReplicaSetをいくつ保持するかを指定するために、Deploymentの`.spec.revisionHistoryLimit`フィールドを設定できます。残りはガベージコレクションされます。デフォルト値は10です。

{{< note >}}
**メモ:** このフィールドを明示的に0に設定すると、Deploymentのすべての履歴がクリーンアップされるため、そのDeploymentはロールバックできなくなります。
{{< /note >}}

## ユースケース {#use-cases}

### Canary Deployment

ロールアウトをDeploymentを使っているユーザやサーバの一部に適用したい場合、複数のDeploymentを作成することができ、片方には[managing resources](/docs/concepts/cluster-administration/manage-deployment/#canary-deployments)のカナリアパターンを記述します。

## Deploymentスペックの記述 {#writing-a-deployment-spec}

他のKubernetes構成と同じく、Deploymentには`apiVersion`、`kind`、`metadata`のフィールドが必要です。構成ファイルの利用についての一般的な情報は、[アプリケーションのデプロイ](/docs/tutorials/stateless-application/run-stateless-application-deployment/)を、コンテナの構成については、[リソース管理のためのkubectlの利用](/docs/concepts/overview/object-management-kubectl/overview/)のドキュメントを参照してください。

Deploymentは[`.spec`セクション](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status)も必要です。

### Podテンプレート {#pod-template}

`.spec.template`は`.spec`の唯一の必須フィールドです。

`.spec.template`は[Podテンプレート](/ja/docs/concepts/workloads/pods/pod-overview/#pod-templates)です。これは、入れ子であることと`apiVersion`や`kind`を持たないこと以外は[Pod](/docs/concepts/workloads/pods/pod/)のスキーマと全く同じです。

Podに対する必須フィールドに加えて、DeploymentのPodテンプレートには適切なラベルと適切な再起動ポリシを指定しなければなりません。ラベルに対しては、他のコントローラと重複しないようにしてください。[セレクタ](#selector)を参照。

[`.spec.template.spec.restartPolicy`](/ja/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)は`Always`のみが指定可能で、指定しない場合はこれがデフォルトです。

### レプリカ数 {#replicas}

`.spec.replicas`は希望するPod数を指定する、オプションのフィールドです。デフォルトは1です。

### セレクタ {#selector}

`.spec.selector`は、このDeploymentの対象となるPodの[ラベルセレクタ](/docs/concepts/overview/working-with-objects/labels/)を指定するオプションのフィールドです。

`.spec.selector`は`.spec.template.metadata.labels`にマッチする必要があり、そうでなければAPIに拒絶されます。

APIバージョン `apps/v1`では、`.spec.selector`と`.metadata.label`のデフォルトが`.spec.template.metadata.labels`となりません。そのため、明示的に設定しなければなりません。`apps/v1`ではDeploymentの作成後、`.spec.selector`はイミュータブルになることにも注意してください。

Deploymentは、セレクタにマッチするラベルを持つPodに対して、`.spec.template`がPodのテンプレートが異なっていたり、Podの総数が`.spec.replicas`を超えていたりすると、そのPodを終了させる可能性があります。また、希望する数よりもPod数が少ないと、`.spec.template`に基づいて、新しいPodを立ち上げます。

{{< note >}}
**メモ:** 直接であれ、他のDeployment経由であれ、ReplicaSetのような他のコントローラ経由であれ、このセレクタにマッチするPodを作成するべきではありません。もしそうすると、最初のDeploymentがこれらのPodを作成したと考えます。Kubernetesはこうすることを止めません。
{{< /note >}}

もしセレクタが重複する複数のコントローラがあると、コントローラ同士で争い、正常な動作をしません。

### 戦略 {#strategy}

`.spec.strategy`は、古いPodを新しいPodで置き換えるときに使う戦略を指定します。`.spec.strategy.type`には、"Recreate"か"RollingUpdate"が指定できます。デフォルトは"RollingUpdate"です。

#### Deploymentの再作成 {#recreate-deployment}

`.spec.strategy.type==Recreate`の場合、新しいPodを作成する前に、既存のPodをすべて削除します。

#### Deploymentのローリングアップデート {#rolling-update-deployment}

`.spec.strategy.type==RollingUpdate`の場合、Deploymentは[ローリングアップデート](/docs/tasks/run-application/rolling-update-replication-controller/)を使ってPodを更新します。ローリングアップデートのプロセスを制御するために、`maxUnavailable`と`maxSurge`を指定できます。

##### Max Unavailable

`.spec.strategy.rollingUpdate.maxUnavailable`は、更新プロセス中で利用不可となるPod数の最大を指定するオプションのフィールドです。絶対数 (例えば5) か希望するPod数のパーセンテージ (例えば10%) で指定できます。パーセンテージからの絶対数は切り捨てで計算されます。`.spec.strategy.rollingUpdate.maxSurge`が0であれば、0を指定することはできません。デフォルトは25%です。

例えば、この値が30%に設定されていた場合、ローリングアップデートが始まると古いReplicaSetは希望するPod数の70%までスケールダウンされます。新しいPodが準備できると、新しいReplicaSetのスケールアップに従って、古いReplicaSetはさらにスケールダウンされます。アップデート全体で希望するPod数の少なくとも70%が利用可能であるよう維持されます。

##### Max Surge

`.spec.strategy.rollingUpdate.maxSurge`は、希望するPod数を超えて作成できるPodの最大数を指定するオプションのフィールドです。値は絶対数 (例えば5) か希望するPod数のパーセンテージ (例えば10%) で指定できます。`MaxUnavailable`が0であれば、この値を0にすることはできません。パーセンテージからの絶対数は切り上げで計算されます。デフォルト値は25%です。

例えば、この値が30%に指定されている場合、新しいReplicaSetはローリングアップデートが始まると、古いPodと新しいPodの総数が希望するPod数の130%になるよう、即座にスケールアップされます。古いPodが削除されると、新しいReplicaSetはさらにスケールアップされ、更新中に実行されるPodの総数が最大で希望するPod数の130%を超えないよう維持されます。

### Progress Deadline Seconds

`.spec.progressDeadlineSeconds`は、システムがDeploymentが、リソースのステータスで`Type=Progressing`、`Status=False`という状態として[失敗した](#failed-deployment)と報告する前に待つ秒数です。DeploymentコントローラはそのDeploymentをリトライし続けます。将来、自動ロールバックが実装されると、Deploymentコントローラはこの条件を観測するとすぐにDeploymentをロールバックするようになるでしょう。

指定する場合、このフィールドは`.spec.minReadySeconds`より大きくする必要があります。

### Min Ready Seconds

`.spec.minReadySeconds`は、新しく作成されたPodがコンテナのクラッシュなしに、Readyから利用可能になると見なされるまでの最小の秒数です。デフォルトは0です (PodはReadyになるとすぐに利用可能であると見なされます)。いつPodがReadyになると見なされるかについての詳細は[コンテナの検査](/ja/docs/concepts/workloads/pods/pod-lifecycle/#container-probes)を参照してください。

### Rollback To

フィールド`.spec.rollbackTo`は、APIバージョン`extensions/v1beta1`と`apps/v1beta1`で非推奨となり、`apps/v1beta2`からはサポートされなくなりました。代わりに、[以前のリビジョンへのロールバック](#rolling-back-to-a-previous-revision)で紹介した`kubectl rollout undo`を使うべきです。

### Revision History Limit

Deploymentのリビジョン履歴は、制御するReplicaSetに格納されます。

`.spec.revisionHistoryLimit`は、ロールバックを行うために残す古いReplicaSetの数を指定するオプションのフィールドです。最良の値は新しいDeploymentの頻度と安定性に依存します。デフォルトでは古いReplicaSetのすべてが保持され、このフィールドを指定しなければ、`etcd`のリソースを消費し、`kubectl get rs`の出力が多くなります。各Deploymentリビジョンの構成はそのReplicaSetに格納されているので、古いReplicaSetが削除されると、そのリビジョンへロールバックできなくなります。

さらに、このフィールドをゼロに設定することは、レプリカ数が0の古いReplicaSetがすべてクリーンアップされることを意味します。この場合、リビジョン履歴がクリーンアップされているので、新しいDeploymentのロールアウトを元に戻すことができません。

### Paused

`.spec.paused`は、Deploymentを停止・再開するためのオプションの真偽値フィールドです。停止しているDeploymentとそうでないDeploymentの違いは、停止しているDeploymentが停止している間は、PodTemplateSpecへの変更によって新しいロールアウトが開始されないということです。作成時、デフォルトでDeploymentは停止されません。

## Deploymentの代替 {#alternative-to-deployments}

### kubectl rolling update

[`kubectl rolling update`](/docs/reference/generated/kubectl/kubectl-commands#rolling-update)は、PodとReplicationControllerを同じような方法で更新します。ですが、Deploymentのほうが、宣言的で、サーバサイドで、ロールバックのような機能があるのでお勧めです。

{{% /capture %}}
