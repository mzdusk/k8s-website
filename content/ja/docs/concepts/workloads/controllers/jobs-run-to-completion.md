---
title: Jobs - Run to Completion
content_template: templates/concept
weight: 70
---

{{% capture overview %}}

_Job_ は1つもしくは複数のPodを作成し、指定した数のPodが正常に終了することを保証します。Podが正常に終了すると、 _Job_ は成功した完了を追跡します。成功した完了数が指定した数に届くと、Jobは自身を完了させます。Jobを削除することで、そのJobが作成したPodをクリーンアップします。

単純なケースは1つのPodを確実に実行するための1つのJobオブジェクトを作成することです。Jobは、最初のPodが失敗するか削除されるか (例えば、ノードのハードウェア障害やノードの再起動など) すると新しいJobをスタートさせます。

Jobは複数のPodを並列で実行させるために使うこともできます。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## Jobの例の実行

Job構成の例を示します。これは円周率を2000の精度で計算し、出力します。完了まで10秒ほどかかります。

{{< code file="job.yaml" >}}

Jobの例を実行します。

```shell
$ kubectl create -f ./job.yaml
job "pi" created
```

次のコマンドでJobの状態をチェックする。

```shell
$ kubectl describe jobs/pi
Name:             pi
Namespace:        default
Selector:         controller-uid=b1db589a-2c8d-11e6-b324-0209dc45a495
Labels:           controller-uid=b1db589a-2c8d-11e6-b324-0209dc45a495
                  job-name=pi
Annotations:      <none>
Parallelism:      1
Completions:      1
Start Time:       Tue, 07 Jun 2016 10:56:16 +0200
Pods Statuses:    0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:       controller-uid=b1db589a-2c8d-11e6-b324-0209dc45a495
                job-name=pi
  Containers:
   pi:
    Image:      perl
    Port:
    Command:
      perl
      -Mbignum=bpi
      -wle
      print bpi(2000)
    Environment:        <none>
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen    LastSeen    Count    From            SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----            -------------    --------    ------            -------
  1m           1m          1        {job-controller }                Normal      SuccessfulCreate  Created pod: pi-dtn4q
```

完了したJobのPodを見るためには `kubectl get pods` を使います。

Jobに属するPodをマシンリーダブルな形で列挙するには、次のようなコマンドが使えます。

```shell
$ pods=$(kubectl get pods --selector=job-name=pi --output=jsonpath={.items..metadata.name})
$ echo $pods
pi-aiw0a
```

ここで、selectorはJobのselectorと同じにします。`--output=jsonpath`オプションは返ってきた一覧の各Podから名前だけを取得する式を指定します。

```shell
$ kubectl logs $pods
3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608640344181598136297747713099605187072113499999983729780499510597317328160963185950244594553469083026425223082533446850352619311881710100031378387528865875332083814206171776691473035982534904287554687311595628638823537875937519577818577805321712268066130019278766111959092164201989380952572010654858632788659361533818279682303019520353018529689957736225994138912497217752834791315155748572424541506959508295331168617278558890750983817546374649393192550604009277016711390098488240128583616035637076601047101819429555961989467678374494482553797747268471040475346462080466842590694912933136770289891521047521620569660240580381501935112533824300355876402474964732639141992726042699227967823547816360093417216412199245863150302861829745557067498385054945885869269956909272107975093029553211653449872027559602364806654991198818347977535663698074265425278625518184175746728909777727938000816470600161452491921732172147723501414419735685481613611573525521334757418494684385233239073941433345477624168625189835694855620992192221842725502542568876717904946016534668049886272327917860857843838279679766814541009538837863609506800642251252051173929848960841284886269456042419652850222106611863067442786220391949450471237137869609563643719172874677646575739624138908658326459958133904780275901
```

## Jobスペックを書く

他のKubernetes構成と同じく、Jobは`apiVersion`, `kind`, `metadata`フィールドを必要とします。

また、Jobは[`.spec`セクション](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status)も必要です。

### Podテンプレート

`.spec.template`は`.spec`で唯一必要なフィールドです。

`.spec.template`は[Podテンプレート](/ja/docs/concepts/workloads/pods/pod-overview/#pod-templates)です。これは、`apiVersion`と`kind`が入れ子にならない以外、Podと全く同じスキーマです。

Podに必要なフィールドに加え、JobのPodテンプレートでは適切なラベル ([Podセレクタ](#podセレクタ)) と適切な再起動ポリシを指定しなければなりません。

[`RestartPolicy`](/ja/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)は`Never`か`OnFailure`のみが許されます。

### Podセレクタ

`.spec.selector`フィールドは任意です。ほぼすべてのケースでこれを指定すべきではありません。[自身のPodセレクタを指定する](#自身のpodセレクタを指定する)を参照してください。

### 並列Job

Jobには主に3つのタイプがあります

1. 非並列Job
  - 通常、Podが失敗しなけば、ただ1つのPodが起動される
  - Podが正常に終了すれば完了となる
2. *固定完了数* の並列Job
  - `.spec.completions`に非ゼロ値を指定する
  - 1から`.spec.completions`の範囲で各値に対して1つの成功したPodがあるとJobは完了となる
  - **未実装:** 各Podは1から.spec.completionsの範囲で異なるインデックスを渡す
3. *ワークキュー* の並列Job:
  - `.spec.completions`を指定せず、`.spec.parallelism`をデフォルトにする
  - Podは何を起動するか決めるために自身もしくは外部サービスと連携しなければならない。
  - 各Podはそのすべてのピアを実行するかどうかを独立に決定できるので、全体のJobは実行される
  - いずれかのPodが正常に終了すると、新しいPodは作成されない
  - 一度、少なくとも1つのPodが正常に終了し、すべてのPodが終了すると、そのJobは正常に完了する
  - 一度いずれかのPodが正常に終了すると、他のPodは動作を続けたり出力をしたりすべきではない。それらはすべて終了するプロセスである

非並列Jobに対して、`.spec.completions`と`.spec.parallelism`は両方とも設定しないでおけます。両方とも設定しないと、どちらもデフォルトの1に設定されます。

固定完了数Jobに対して、必要な完了数を`.spec.completions`に設定すべきです。`.spec.parallelism`も設定できるが、設定しなければデフォルトの1となります。

ワークキューJobに対して、`.spec.completions`は設定してはならず、`.spec.parallelism`に非負の整数を設定しなければなりません。

異なるタイプのJobの活用方法についての詳細は[Jobパターン](#jobパターン)の節を参照してください。

#### 並列度の制御

要求される並列度 (`.spec.parallelism`) にはあらゆる非負の値を設定できます。指定しなければ、デフォルトの1となります。0に設定すると、そのJobは値が増やされるまで効率的に停止されます。

実際の並列度 (あらゆるインスタンスで実行するPodの数) は様々な理由で、要求される並列度よりも多くなったり少なくなったりします。

- 固定完了数Jobに対して、実際に並列で実行しているPodの数は残りの完了数を超えない
- ワークキューJobに対して、いずれかのPodが成功した後に新しいPodは開始されない -- しかし、残りのPodは完了できる
- コントローラが反応する時間がない場合
- 何らかの理由 (ResourceQuotaの不足や、権限の不足など) でコントローラがPodの作成に失敗すると、要求よりもPodが少なくなる可能性がある
- 同じJobで前のPodの失敗が多いために、コントローラが新しいPodの作成を抑制する可能性がある
- Podが但しくシャットダウンする際に、停止まで時間がかかる

## Podとコンテナの失敗の扱い

Pod内のコンテナは中のプロセスが非ゼロの終了コードで終了したり、メモリ制限を超えたためにkillされたりするなど、多くの理由で失敗する可能性があります。これが発生し、`.spec.template.spec.restartPolocy = "OnFailure"`であると、Podはノードに留まりますが、コンテナは再実行されます。したがって、プログラムはローカルで再起動された場合に対応する必要があります。もしくは`spec.template.spec.restartPolicy = "Never"`を指定します。`restartPolocy`の詳細は[Podの状態](/docs/concepts/workloads/pods/pod-lifecycle/#example-states)を参照してください。

また、(ノードのアップグレードや再起動、削除などによって) Podがノードから追い出されたり、Podのコンテナが失敗し、`.spec.template.spec.restartPolicy = "Never"`であったりするなど様々な理由でPod全体が失敗することもあります。Podが失敗すると、Jobコントローラは新しいPodを開始します。したがって、プログラムは新しいPodで再起動する場合に対応する必要があります。特に、一時ファイルやロック、中途半端な出力、前回の実行で発生したものに対応する必要があります。

`.spec.parallelism = 1`と、`.spec.completions = 1`、`.spec.template.spec.restartPolicy = "Never"`を指定していたとしても、同じプログラムが時々2回起動されることに注意してください。

`.spec.parallelism`と`.spec.completions`を両方ともを1より大きい値に指定している場合、複数のPodが同時に実行される可能性があります。したがって、Podは同時実行にも対応しなければなりません。

### Podのバックオフ失敗ポリシ

構成などでの論理エラーのためのリトライ数がある程度増えた場合にJobを失敗させたい状況があります。そうするために、`.spec.backoffLimit`にJobを失敗とみなすリトライ数を指定します。バックオフリミットはデフォルトでは6に設定されます。Jobに関連した失敗したPodは、6分を限度とする指数的なバックオフディレイ (10s, 20s, 40s) の後、Jobコントローラによって再作成されます。Jobの次のステータスチェックの際に新たに失敗したPodがなければ、バックオフカウントはリセットされます。

{{< note >}}
**メモ:** 既知の問題 [#54870](https://github.com/kubernetes/kubernetes/issues/54870) により、`.spec.template.spec.restartPolicy`フィールドに"`OnFailure`"が設定されると、バックオフ制限が無効になる可能性があります。手っ取り早い回避策は、再起動ポリシに"`Never`"を設定することです。
{{< /note >}}

## Jobの終了とクリーンアップ

Jobが完了すると、それ以上のPodは作成されませんが、削除もされません。それらを保持しておくことで、エラーや警告、その他診断用の出力をチェックするために完了したPodのログを見られるようになります。完了した後のJobオブジェクトも残るのでそのステータスも見ることができます。Jobは`kubectl`で削除します (例えば `kubectl delete jobs/pi` もしくは `kubectl delete -f ./job.yaml`)。`kubectl`を使ってJobを削除すると、作成されたPodも削除されます。

デフォルトでは、JobはPodが失敗しなければ連続して実行し、Podが失敗すれば上で述べた`.spec.backoffLimit`に従います。Jobを終了するその他の方法は有効な期限を設定することです。これはJobの`.spec.activeDeadlineSeconds`フィールドに秒数を設定することで行います。

`activeDeadlineSeconds`はどれだけ多くのPodが作成されたとしても、Jobの長さに適用されます。一度Jobが`activeDeadlineSeconds`に到達すると、JobとそのすべてのPodは終了されます。その結果、Jobのステータスは`reason: DeadlineExceeded`となります。

Jobの`.spec.acriveDeadlineSeconds`は`.spec.backoffLimit`よりも優位であることに注意してください。したがって、1つもしくは複数の失敗PodをリトライするJobは、一度`activeDeaflineSeconds`で指定したタイムリミットに到達すると、`backoffLimit`に到達していなかったとしても、追加のPodをデプロイしません。

例:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-timeout
spec:
  backoffLimit: 5
  activeDeadlineSeconds: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

Job内のJobスペックと[Podテンプレートスペック](/ja/docs/concepts/workloads/pods/init-containers/#detailed-behavior)の両方ともが`activeDeadlineSeconds`フィールドを持つことに注意してください。適切なレベルにこのフィールドを設定していることを確認してください。

## Jobパターン {#job-patterns}

JobオブジェクトはPodの信頼できる並列実行をサポートするために使われます。Jobオブジェクトは、科学計算で一般に見られるような、密接に通信する並列プロセスをサポートするようには設計されていません。独立であるが関連のあるワークアイテムの集合の並列実行をサポートします。これらは送信されるメールやレンダリングされるフレーム、変換されるファイル、NoSQLデータベースでスキャンするキーの範囲などかもしれません。

複雑なシステムでは、複数の異なるワークアイテムの集合かもしれません。ここでは、ユーザが一緒に管理したい思うワークアイテムの集合、 *バッチジョブ* について考えてみます。

並列計算にはさまざまな異なるパターンがあり、それぞれに長所と短所があります。そのトレードオフは次のようなものです。

- 各ワークアイテムごとのJobオブジェクト 対 全てのワークアイテムを含む単一Jobオブジェクト。大量のワークアイテムに対しては後者のほうが良い。前者は大量のJobオブジェクトを管理するためにユーザとシステムに対していくぶんかのオーバヘッドができる。
- ワークアイテムと同数のPodを作成 対 各Podが複数のワークアイテムを処理。前者は概して既存コードとコンテナの修正が少なく済む。前項の理由から、大量のワークアイテムに対しては後者のほうが良い。
- いくつかのアプローチはワークキューを使う。これにはキューサービスを実行し、既存のプログラムやコンテナをそのキューを使うように修正する必要がある。その他のアプローチは既存のコンテナ化したアプリケーションに適合しやすい。

ここにそのトレードオフをまとめます。

|                            パターン                                   | 単一Jobオブジェクト | ワークアイテムよりPodが少ない? | 修正なしのアプリを使う? |  Kube 1.1で動作する? |
| -------------------------------------------------------------------- |:-----------------:|:---------------------------:|:-------------------:|:-------------------:|
| [Job Template Expansion](/ja/docs/tasks/job/parallel-processing-expansion/)            |                   |                             |          ✓          |          ✓          |
| [Queue with Pod Per Work Item](/ja/docs/tasks/job/coarse-parallel-processing-work-queue/)   |         ✓         |                             |      時々      |          ✓          |
| [Queue with Variable Pod Count](/ja/docs/tasks/job/fine-parallel-processing-work-queue/)  |         ✓         |             ✓               |                     |          ✓          |
| Single Job with Static Work Assignment                               |         ✓         |                             |          ✓          |                     |

`.spec.completions`に完了数を指定すると、Jobコントローラによって作成された各Podは同じ[`spec`](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status)を持ちます。これはすべてのPodが同じコマンドラインと同じイメージ、同じボリューム、(ほとんど)同じ環境変数を持つことを意味します。これらのパターンはPodが異なることを行うための準備をする異なる方法です。

次の表は各パターンに対して`.spec.parallelism`と`.spec.completions`に必要な設定を示します。`W`はワークアイテムの数です。

|                             パターン                                  | `.spec.completions` |  `.spec.parallelism` |
| -------------------------------------------------------------------- |:-------------------:|:--------------------:|
| [Job Template Expansion](/ja/docs/tasks/job/parallel-processing-expansion/)           |          1          |     1であるべき      |
| [Queue with Pod Per Work Item](/ja/docs/tasks/job/coarse-parallel-processing-work-queue/)   |          W          |        何でもよい           |
| [Queue with Variable Pod Count](/ja/docs/tasks/job/fine-parallel-processing-work-queue/)  |          1          |        何でもよい           |
| Single Job with Static Work Assignment                               |          W          |        何でもよい           |

## 高度な使い方

### 自身のPodセレクタを指定する

通常、Jobオブジェクトを作成するときに`.spec.selector`は指定しません。Jobが作られる時にシステムのデフォルトロジックがこのフィールドを追加します。これはほかのJobと重複しないセレクタ値を選びます。

しかしながら、あるケースでは、この動作を上書きする必要があるかもしれません。そうするために、Jobの`.spec.selector`を指定することができます。

これを行う際には特に注意してください。一意でないラベルセレクタを指定し、関係ないPodにマッチすると、関係ないJobのPodが削除されたり、他のPodを完了したものとして数えたり、JobがPodの作成や実行を拒否したりする可能性があります。一意でないセレクタが選ばれると、他のコントローラとそのPodも予測できないふるまいをするかもしれません。Kubernetesは`.spec.selector`を指定するときのミスを止めることはできません。

この機能を使いたいと思う場合の例を挙げます。

`old`というJobがすでに実行しているとします。既存のPodは実行させておきたいが、それ以降に作成されるPodには異なるPodテンプレートを使わせ、Jobには新しい名前を持たせたい。これらのフィールドは更新不可のためJobを更新することはできません。そのため、`kubectl delete jobs/old --cascade=false`を使い、Podを実行させたまま`old` Jobを削除します。削除する前に、どのセレクタを使っていたのかを記録しておきます。

```yaml
kind: Job
metadata:
  name: old
  ...
spec:
  selector:
    matchLabels:
      job-uid: a8f3d00d-c6d2-11e5-9f87-42010af00002
  ...
```

次に、newという名前の新しいJobを作成し、明示的に同じセレクタを指定します。既存のPodは`job-uid=a8f3d00d-c6d2-11e5-9f87-42010af00002`というラベルを持っているので、`new` Jobにより制御されます。

システムが自動生成するセレクタを使っていないため、新しいJobでは`manualSelector: true`を指定する必要があります。

```yaml
kind: Job
metadata:
  name: new
  ...
spec:
  manualSelector: true
  selector:
    matchLabels:
      job-uid: a8f3d00d-c6d2-11e5-9f87-42010af00002
  ...
```

新しいJob自身は `a8f3d00d-c6d2-11e5-9f87-42010af00002` とは異なるUIDを持ちます。`manualSelector: true`に設定することでシステムに何をしているか知っていて、このミスマッチを許容するように伝えます。

## 代替手段

### Bare Pod

Podを実行しているノードが再起動もしくは故障した場合、Podは終了し再起動されません。しかしながら、Jobは終了したPodを置き換えるために新しいPodを作成します。このため、アプリケーションで単一のPodしか必要ないとしても、Bare PodよりもJobを使うことを推奨します。

### Replication Controller

JobはReplication Controllerを補完するものです。[Replication Controller](/ja/docs/user-guide/replication-controller)は (Webサーバのような) 終了することを期待しないPodを管理し、Jobは(バッチジョブのような)終了することを期待するPodを管理します。

[Podのライフサイクル](/ja/docs/concepts/workloads/pods/pod-lifecycle/)で議論したように、`Job`は`RestartPolicy`が`OnFailure`または`Never`であるPodに *のみ* 適している。(注: `RestartPolicy`が設定されなければ、デフォルト値は`Always`である。)

### 単一JobがController Podを起動する

その他のパターンは、カスタムコントローラのように動作する他のPodを起動するためのPodを作成する単一Jobのためのものです。これは最も柔軟ですが、開始するのがいくらか複雑で、Kubernetesとの統合がより少なくなります。

このパターンの一例は順にSparkのマスタコントローラを起動し ([sparkの例](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/spark/README.md)を参照)、Sparkドライバを実行し、クリーンアップするスクリプトを実行するPodを自動するJobです。

このアプローチの利点は、プロセス全体が完了することを保証することですが、どのPodが作られるかやどのように動作するかといった完全な制御はそのPodにゆだねられます。

## Cron Job

指定した日時でのJobの作成のサポートはKubernetes [1.4](https://github.com/kubernetes/kubernetes/pull/11980)で利用可能になりました。詳細は[Cron Jobのドキュメント](/ja/docs/concepts/workloads/controllers/cron-jobs/)を参照してください。

{{% /capture %}}
