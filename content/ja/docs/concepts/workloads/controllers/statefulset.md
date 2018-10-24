---
title: StatefulSets
content_template: templates/concept
weight: 40
---

{{% capture overview %}}

StatefulSetはステートフルアプリケーションを管理するために使うワークロードAPIオブジェクトです。

{{< note >}}
**メモ:** StatefulSetは1.9で安定 (GA) になりました。
{{< /note >}}

{{< glossary_definition term_id="statefulset" length="all" >}}
{{% /capture %}}

{{% capture body %}}

## StatefulSetを使う {#using-statefulsets}

StatefulSetは以下の1つ以上を必要とするアプリケーションに価値があります。

* 安定した一意のネットワークID
* 安定した永続ストレージ
* 順序のある、きちんとしたデプロイとスケーリング
* 順序のある、自動化されたローリングアップデート

上で、安定とはPodの(再)スケジュールを通じて永続することと同義です。もしアプリケーションが安定したIDや順序付きデプロイ、削除、スケーリングを必要としないのなら、ステートレスレプリカの集合を提供するコントローラでデプロイすべきです。[Deployment](/ja/docs/concepts/workloads/controllers/deployment/)や[ReplicaSet](/ja/docs/concepts/workloads/controllers/replicaset/)のようなコントローラはステートレスなニーズにより向いています。

## 制限 {#limitations}

* StatefulSetは1.9より前ではベータリソースで、1.5より前のリリースでは利用できません
* Podに与えられるストレージは要求されるstorage classに基づいた[PersistentVolume Provisioner](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/persistent-volume-provisioning/README.md)によって供給されるか、管理者によって前もって供給されなければなりません
* StetefulSetの削除やスケールダウンはそのStatefulSetに関連するボリュームを削除しません。これはデータの安全性を保証するためであり、一般的に関連するStatefulSetリソースの自動削除よりも価値があります。
* StatefulSetは現在PodのネットワークIDに責任を負うため[Headless Service](/ja/docs/concepts/services-networking/service/#headless-services)が必要です。このサービスを作る責務はユーザにあります。
* StatefulSetは、StatefulSetが削除された場合にPodが終了することを保証しません。きちんと順序だった削除を行うためには、StatefulSetを削除する前にスケールを0にします。

## コンポーネント {#components}

以下にStatefulSetのコンポーネントを例示します。

* nginxという名前のHeadless Serviceはネットワークドメインを制御するために使われます
* webという名前のStatefulSetは3レプリカのnginxコンテナが別々のPodとして起動するよう指示するスペックを持ちます
* volumeClaimTemplateはPersistentVolume Provisionerによって供給される[PersistentVolume](/ja/docs/concepts/storage/persistent-volumes/)を使う安定したストレージを提供します

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # .spec.template.metadata.labelsにマッチする必要があります
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # .spec.selector.matchLabelsにマッチする必要があります
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

## Podセレクタ {#pod-selector}

StatefulSetには`.spec.template.metadata.labels`のラベルと一致する`.spec.selector`フィールドを設定しなければなりません。Kubernetes 1.8より前は、`.spec.selector`が省略されると、デフォルト値が使われました。1.8以降ではマッチするPodセレクタの指定に失敗すると、StatefulSet作成中に不整合エラーが発生します。

### Pod ID {#pod-identity}

Stateful Podは順の安定したネットワークIDと安定したストレージから成る一意のIDを持ちます。このIDはどのノードに(再)スケジュールされたかにかかわらず永続的です。

### 順序インデックス {#ordered-index}

NレプリカのStatefulSetに対して、StatefulSetの各Podは0からN-1まで順に整数が割り当てられます。

### 安定したネットワークID {#stable-network-id}

StatefulSetの各PodはStaetfulSetの名前とPodの順からホスト名を得ます。構築されるホスト名のパターンは`$(statefulset name)-$(ordinal)`です。上の例では、`web-0, web-1, web-2`という名前の3つのPodが作られます。StatefulSetはそのPodのドメインを制御するために[Headless Service](/ja/docs/concepts/services-networking/service/#headless-services)を使えます。このサービスによって管理されるドメインは `$(service name).$(namespace).svc.cluster.local`という形をとり、"cluster.local"はクラスタドメインです。各Podが作成されると、マッチするDNSサブドメインを取得し、これは`$(podname).$(govering service domain)`という形をとり、govering serviceはStatefulSetの`serviceName`によって定義されます。

クラスタドメイン、サービス名、StatefulSet名、そしてStatefulSetのPodに対するDNS名がどのように適用されるかの選択例を示します。

クラスタドメイン | サービス (ns/name) | StatefulSet (ns/name)  | StatefulSetドメイン  | Pod DNS | Podホスト名 |
-------------- | ----------------- | ----------------- | -------------- | ------- | ------------ |
 cluster.local | default/nginx     | default/web       | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |
 cluster.local | foo/nginx         | foo/web           | nginx.foo.svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local     | web-{0..N-1} |
 kube.local    | foo/nginx         | foo/web           | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local        | web-{0..N-1} |

[何も構成しない](/ja/docs/concepts/services-networking/dns-pod-service/#how-it-works)場合、クラスタドメインは`cluster.local`に設定されることに注意してください。

### 安定したストレージ {#stable-storage}

Kubernetesは各VolumeClaimTemplateに対して1つの[PersistentVolume](/ja/docs/concepts/storage/persistent-volumes/)を作成します。上のnginxの例では、各Podが`my-storage-class`のStorageClassと1Gibの供給されたストレージを持つ単一のPersistentVolumeを受け取ります。StorageClassは指定されなければ、デフォルトのStorageClassが使われます。Podがノードに(再)スケジュールされると、その`volumeMounts`はそのPersistentVolumeClaimに関連するPersistentVolumeをマウントします。PodやStatefulSetが削除された時に、PodのPersistentVolumeClaimに関連するPersistentVolumeは削除されないことに注意してください。これは手動で削除しなければなりません。

### Pod名ラベル {#pod-name-label}

StatefulSetコントローラがPodを作成するとき、Podの名前を設定する`statefulset.kubernetes.io/pod-name`ラベルを追加します。

## デプロイとスケーリング保証 {deployment-and-scaling-guarantees}

* NレプリカのStatefulSetに対して、Podがデプロイされると、0からN-1の順で順次作成されます。
* Podが削除されると、N-1から0の逆順に終了します。
* スケーリング操作がPodに適用される前に、先行するすべてのPodはRunning and Readyでなければなりません。
* Podが終了する前に、後継となるPodはすべて完全にシャットダウンされていなければなりません

StatefulSetは`pod.Spec.TerminationGracePeriodSeconds`を0に指定すべきではない。これは安全でなく、強く推奨しません。さらなる説明は[StatefulSet Podの強制削除](/ja/docs/tasks/run-application/force-delete-stateful-set-pod/)を参照してください。

上のnginxの例が作られる時、3つのPodがweb-0, web-1,web-2の順にデプロイされる。web-1はweb-0が[Running and Ready](/ja/docs/user-guide/pod-states/)になる前にデプロイされることはなく、web-2はweb-1がRunning and Readyになる前にデプロイされることはありません。web-1がRunning and Readyになり、web-2か起動する前にweb-0が失敗したとしても、web-2はweb-0が正常に起動しRunning and Readyになるまで起動されません。

ユーザがStatefulSetに`replicas=1`のようなパッチを当てることでデプロイされた例をスケールすると、web-2が最初に終了されます。web-1はweb-2が完全にシャットダウンし削除されるまで終了されません。web-2が終了して完全にシャットダウンし、web-1が終了する前にweb-0が失敗すると、web-1はweb-0がRunning and Readyになるまで終了されません。

### Pod管理ポリシ {#pod-management-policies}

Kubernetes 1.7以降では、StatefulSetは`.spec.podManagementPolicy`フィールドを通じて一意性と同一性の保証を予約することで順序保証を緩めることができます。

#### OrderedReady Pod管理 {#orderedready-pod-management}

`OrderedReady` Pod管理はStatefulSetのデフォルトである。これは[上記](#デプロイとスケーリング保証)のようなふるまいを実装しています。

#### Parallel Pod管理 {#parallel-pod-management}

`Parallel` Pod管理はStatefulSetコントローラに全てのPodを並列に起動もしくは終了するよう伝えます。これは前のPodの状態がRunning and Readyもしくは完全に終了するまで待つことはありません。

## 更新戦略 {#update-strategies}

Kubernetes 1.7以降、StatefulSetの`.spec.updateStrategy`フィールドでコンテナやラベル、リソース要求/制限、Podのアノテーションに対して自動ローリングアップデートを構成または無効にできるようになりました。

### On Delete

`OnDelete`更新戦略は昔の(1.6以前)ふるまいを実装しています。StatefulSetの`.spec.updateStrategy.type`を`OnDelete`に設定すると、StatefulコントローラはStatefulSetのPodを自動で更新しなくなります。StatefulSetの`.spec.template`に行った更新を反映した新しいPodをコントローラに作成させるためにはユーザが手動でPodを削除しなければなりません。

### ローリングアップデート {#rolling-updates}

`RollingUpdate`更新戦略はStatefulSetのPodを自動でローリングアップデートをする実装です。これは`.spec.updateStrategy`が指定されなかった場合のデフォルト戦略です。StatefulSetの`.spec.updateStraetegy.type`を`RollingUpdate`に設定すると、StatefulControllerはStatefulSetのPodを削除し、再作成するようになります。Podの終了は同じ順 (最大の数字から最小へ) で実行され、Podの更新は一度に1つずつ行われます。更新されたPodがRunning and Readyになるまで次の更新は待ちます。

#### 分割 {#partitions}

`RollingUpdate`更新戦略は`.spec.updateStrategy.rollingUpdate.partition`を指定することで分割できます。もし分割が指定されると、分割以上の数字を持つすべてのPodがStatefulSetの`.spec.template`が更新されたときに更新されます。分割未満の数字を持つPodは、それらが削除されたとしても更新されず、前のバージョンで再作成されます。StatefulSetの`.spec.updateStrategy.rollingUpdate.partition`が`.spec.replicas`より大きい場合、`.spec.template`の更新はPodに伝わりません。ほとんどの場合分割を使うことはありませんが、更新をステージしたり、テスト版をロールアウトしたり、段階的にロールアウトを実行したりしたい場合には便利です。

{{% /capture %}}
{{% capture whatsnext %}}

* [ステートフルアプリケーションのデプロイ](/ja/docs/tutorials/stateful-application/basic-stateful-set/)の例に進みます
* [StatefulSetを使ったCassandraのデプロイ](/docs/tutorials/stateful-application/cassandra/)の例に進みます

{{% /capture %}}
