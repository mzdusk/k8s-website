---
reviewers:
title: Initコンテナ
content_template: templates/concept
weight: 40
---

{{% capture overview %}}
このページではInitコンテナの概要を提供します。これはアプリコンテナの前に実行し、アプリイメージには存在しないユーティリティやセットアップスクリプトを含められる特別なコンテナです。
{{% /capture %}}

{{< toc >}}

この機能は1.6でベータから抜けました。Initコンテナは`containers`配列と並べてPodSpecで指定できます。ベータアノテーションの値はまだ考慮され、PodSpecフィールドの値を上書きしますが、1.6と1.7で非推奨になりました。1.8ではこのアノテーションはサポートされず、PodSpecフィールドに変換しなければなりません。

{{% capture body %}}

## Initコンテナを知る

[Pod](/docs/concepts/workloads/pods/pod-overview/)はアプリを実行する複数のコンテナを持てますが、1つ以上のInitコンテナを持つこともでき、これはアプリコンテナがスタートする前に実行されます。

Initコンテナは以下を除いて通常のコンテナと良く似ています。

* 常に完了する
* それぞれは次がスタートする前に正常に完了しなければならない

PodのInitコンテナが失敗すると、KubernetesはInitコンテナが成功するまで繰り返しPodを再起動します。しかしながら、Podの`restartPolicy`がNeverであれば再起動されません。

コンテナをInitコンテナとして指定するためには、アプリの`containers配列`と並列に`initContainers`フィールドをPodSpecに[Container](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#container-v1-core)型オブジェクトのJSON配列として追加します。initコンテナの状態は`.status.initContainerStatuses`フィールドに(`.status.containerStatuses`フィールドと同様の)コンテナ状態の配列として返されます。

### 通常のコンテナとの違い

Initコンテナはリソース制限やボリューム、セキュリティ設定といった、アプリコンテナのすべてのフィールドと機能をサポートします。しかしながら、Initコンテナに対するリソース要求と制限はわずかに異なった扱いとなり、以下の[リソース](#リソース)で詳述します。また、Initコンテナは、PodがReadyとなる前に完了しなければならないため、Readiness検査をサポートしません。

複数のInitコンテナがPodで指定されれば、それらのコンテナはシーケンシャルな順で一度に1つずつ実行されます。それぞれは次が実行される前に成功しなければなりません。すべてのInitコンテナが完了すると、KubernetesはPodを初期化し、通常通りアプリケーションコンテナを実行します。

## Initコンテナは何のために使うのか

Initコンテナはアプリコンテナと分離したイメージであるため、スタートアップに関連するコードに対していくつかの利点があります。

* セキュリティ的な理由でアプリコンテナイメージに含めるのが適切でないユーティリティを含めて実行することができる
* アプリイメージに存在しないセットアップのためのユーティリティやカスタムコード含められる。例えば、セットアップ中に`sed`や`awk`, `python`, `dig`といったツールを使うためだけの他のイメージを`FROM`に指定したイメージを作成する必要はない。
* アプリケーションイメージ作成者とデプロイ担当者のロールが、単一のアプリイメージを結合してビルドする必要なく独立に働くことができる。
* Linux名前空間を使うので、アプリコンテナとは違うファイルシステムビューを持つ。その結果、アプリコンテナがアクセスできないSecretへのアクセス権を与えることができる
* あらゆるアプリコンテナがスタートする前に動作を完了する一方、アプリコンテナは並列で実行するので、Initコンテナはいくつかの前提条件を満たすまでアプリコンテナのスタートアップをブロックまたは遅延させるための方法を容易に提供できる

### 例

Initコンテナの使い方についてのいくつかのアイデアを挙げます。

* 次のようなshellコマンドでサービスが作成されるのを待つ

      for i in {1..100}; do sleep 1; if dig myservice; then exit 0; fi; done; exit 1

* 次のようなコマンドでこのPodを下流のAPIからリモートサーバへ登録する

      curl -X POST http://$MANAGEMENT_SERVICE_HOST:$MANAGEMENT_SERVICE_PORT/register -d 'instance=$(<POD_NAME>)&ip=$(<POD_IP>)'

* アプリコンテナが起動する前に `sleep 60` のようなコマンドを使っていくらか待つ
* ボリュームにgitリポジトリをcloneする
* 構成ファイルに値を配置し、メインアプリコンテナに対する構成ファイルを動的に生成するためのテンプレートプールを起動する。例えば、構成ファイルのPOD_IPの値を配置し、Jinjaを用いてメインアプリの構成ファイルを生成する。

詳細な使用例は [StatefulSetのドキュメント](/docs/concepts/workloads/controllers/statefulset/)や[本番Podガイド](/docs/tasks/configure-pod-container/configure-pod-initialization/)で見られます。

### 使用中のInitコンテナ

以下のKubernetes 1.5用のyamlファイルは2つのInitコンテナを持つシンプルなPodの要点を述べます。1つ目は`myservice`を待ち、2つ目は`mydb`を待ちます。両方のコンテナが完了すると、Podが開始されます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
  annotations:
    pod.beta.kubernetes.io/init-containers: '[
        {
            "name": "init-myservice",
            "image": "busybox",
            "command": ["sh", "-c", "until nslookup myservice; do echo waiting for myservice; sleep 2; done;"]
        },
        {
            "name": "init-mydb",
            "image": "busybox",
            "command": ["sh", "-c", "until nslookup mydb; do echo waiting for mydb; sleep 2; done;"]
        }
    ]'
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
```

古いannotation文法は1.6と1.7でも動作しますが、Kubernetes 1.6で新しい文法が導入されました。1.8以上では新しい文法を使わなければなりません。Initコンテナの定義をspecに移動しました。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

1.5文法は1.6でも動作しますが、1.6文法を使うことを推奨します。Kubernetes 1.6では、InitコンテナはAPIでフィールドが作られます。betaアノテーションは1.6と1.7では配慮されますが、1.8以降ではサポートされません。

以下のYamlは`mydb`と`myservice`の大枠を記述したものです。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
kind: Service
apiVersion: v1
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
```

このPodは以下のコマンドで開始、デバッグできます。

```shell
$ kubectl create -f myapp.yaml
pod/myapp-pod created
$ kubectl get -f myapp.yaml
NAME        READY     STATUS     RESTARTS   AGE
myapp-pod   0/1       Init:0/2   0          6m
$ kubectl describe -f myapp.yaml
Name:          myapp-pod
Namespace:     default
[...]
Labels:        app=myapp
Status:        Pending
[...]
Init Containers:
  init-myservice:
[...]
    State:         Running
[...]
  init-mydb:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Containers:
  myapp-container:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Events:
  FirstSeen    LastSeen    Count    From                      SubObjectPath                           Type          Reason        Message
  ---------    --------    -----    ----                      -------------                           --------      ------        -------
  16s          16s         1        {default-scheduler }                                              Normal        Scheduled     Successfully assigned myapp-pod to 172.17.4.201
  16s          16s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulling       pulling image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulled        Successfully pulled image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Created       Created container with docker id 5ced34a04634; Security:[seccomp=unconfined]
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Started       Started container with docker id 5ced34a04634
$ kubectl logs myapp-pod -c init-myservice # Inspect the first init container
$ kubectl logs myapp-pod -c init-mydb      # Inspect the second init container
```

一度`mydb`と`myservice` Serviceをスタートさせれば、Initコンテナが完了し、`myapp-pod`が作成される様子がわかります。

```shell
$ kubectl create -f services.yaml
service/myservice created
service/mydb created
$ kubectl get -f myapp.yaml
NAME        READY     STATUS    RESTARTS   AGE
myapp-pod   1/1       Running   0          9m
```

この例はとてもシンプルですが、Initコンテナを作成するための刺激を与えてくれるはずです。

## 詳細なふるまい

Podのスタートアップ中、Initコンテナは、ネットワークとボリュームが初期化された後、順番にスタートします。各コンテナは次が始まる前に正常に終了しなければなりません。コンテナがスタートするのに失敗したり異常終了した場合、Podの`restartPolicy`に応じてリトライします。しかし、Podの`restartPolicy`がAlwaysに設定されている場合、InitコンテナはOnFailureの`RestartPolicy`を使います。

すべてのInitコンテナが成功するまで、Podは`Ready`にはなりません。Initコンテナのポートはサービス下に統合されません。初期化中のPodは`Pending`状態ですが、`Initializing`状態がTrueに設定されているはずです。

Podが[再起動](#podが再起動する理由)されると、すべてのInitコンテナは再び実行しなければなりません。

Initコンテナスペックへの変更はコンテナイメージフィールドに制限されています。Initコンテナイメージフィールドを変更することはPodを再起動することと等価です。

Initコンテナは再起動したり、リトライしたり、再実行される可能性があるので、Initコンテナのコードは冪等であるべきです。特に`EmptyDirs`にファイルを書き込むコードは既に出力ファイルが存在する可能性に留意すべきです。

Initコンテナはアプリコンテナのすべてのフィールドを持ちます。しかしながら、Initコンテナはreadinessと完了の区別を定義できないため、Kubernetesは`readinessProbe`の使用を禁止します。これはバリデーションの時に強制されます。

Initコンテナが永遠に失敗するのを防ぐために、Podの`activeDeadlineSeconds`とコンテナの`livenessProbe`を使います。有効期限はInitコンテナを含みます。

PodのアプリコンテナとInitコンテナの名前は一意でなければなりません。名前が重複する場合、バリデーションエラーが発生します。

### リソース

Initコンテナに与えられた順序と実行に対して、リソースの使用には以下のルールが適用されます。

* すべてのInitコンテナで定義されたリソース要求または制限の最大値が *実効初期要求/制限* である
* リソースに対するPodの *実効要求/制限* は以下より大きい
  * リソースに対するすべてのアプリコンテナの要求/制限の合計
  * リソースに対する実効初期要求/制限
* スケジューリングは実効要求/制限を基に行われ、これはInitコンテナが、Podの生存期間中に使われない初期化のためのリソースを予約できることを意味する
* Podの *実効QoSティア* のQoSティアはInitコンテナとアプリコンテナと同様のQoSティアである

クオータと制限は実効Pod要求と制限に基づいて適用されます。

Podレベルのcgroupはスケジューラと同じく、実効Pod要求と制限に基づきます。

### Podが再起動する理由

以下のような理由で、Initコンテナの再実行が発生してPodが再起動することがあります。

* Initコンテナイメージの変更を起こすようにユーザがPodSpecを更新する。アプリコンテナイメージの変更はアプリコンテナのみを再起動する。
* Podインフラコンテナが再起動される。これは稀であり、誰かがノードへのルートアクセスを行う必要がある。
* `restartPolicy`がAlwaysに設定されている状態でPodのすべてのコンテナが終了し、強制的に再起動され、ガベージコレクションによりInitコンテナの完了記録が失われる。

## サポートと互換性

Apiserverのバージョン1.6.0以上のクラスタは`.spec.initContainers`フィールドを使うInitコンテナをサポートします。それ以前のバージョンはアルファまたはベータアノテーションを使うInitコンテナをサポートします。`.spec.initContainers`フィールドはアルファとベータのアノテーションも反映するので、Kubernetes 1.3.0以上でInitコンテナを実行でき、バージョン1.6の apiserverは既存の作成されたPodに対するInitコンテナ機能を失うことなくバージョン1.5に安全にロールバックできます。

バージョン1.8.0以上のapiserverとkubeletで、アルファとベータのアノテーションに対するサポートは削除され、非推奨のアノテーションを`.spec.initContainers`フィールドに変換する必要があります。

{{% /capture %}}


{{% capture whatsnext %}}

* [Initコンテナを持つPodの作成](/ja/docs/tasks/configure-pod-container/configure-pod-initialization/#creating-a-pod-that-has-an-init-container)

{{% /capture %}}
