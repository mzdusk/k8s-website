---
title: リソースクオータ
content_template: templates/concept
weight: 10
---

{{% capture overview %}}

いくつかのユーザやチームが、固定された数のノードで構成されるクラスタを共有する場合、
1つのチームが共有リソースを大量に使ってしまうことが考えられます。

リソースクオータは、この問題を解決するための管理者向けツールです。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

`ResourceQuota`オブジェクトで定義されるリソースクオータは、名前空間ごとに合計の
リソース消費量を制限する機能を提供します。プロジェクトでリソースごとに消費する
計算リソースと同様に、タイプごとに名前空間内に作成できるオブジェクトの量を制限できます。

リソースクオータは次のように動作します。

- 異なる名前空間で異なるチームが作業をします。現在これは任意ですが、ACL経由で必須にする
  ことをサポートする予定です。
- 管理者は、各名前空間に1つまたは複数の`ResourceQuotas`を作成します。
- ユーザがその名前空間でリソース (PodやServiceなど) を作成し、クオータシステムは、
  `ResourceQuota`で定義したハードリソースリミットを超えないよう使用量を追跡します。
- リソースの作成や更新がクオータ制限に違反するのであれば、そのリクエストは、制限に違反する
  ことを説明するメッセージとともにHTTPステータスコード`403 FORBIDDEN`で失敗します。
- `cpu`や`memory`のような計算リソースに対してクオータが有効であれば、ユーザはこれらの値に
  対して、requestまたはlimitを指定しなければなりません。そうしなければ、クオータシステムは
  Podの作成を拒絶します。ヒント: 計算リソースの指定がないPodに対して、デフォルト値を強制する
  ためには、`LimitRanger`アドミッションコントローラを使ってください。この問題を回避する
  方法の例は
  [ウォークスルー](/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)
  を参照してください。

名前空間とクオータを使って作成するポリシの例は次のようになります。

- 32 GiBのRAMと16コアあるクラスタの場合、チームAに20 GiBと10コアを使わせ、チームBに10 GiBと
  4コアを使わせ、2 GiBと2コアは将来の割り当てのために残しておきます。
- "testing"名前空間は1コアと1 GiBのRAMに制限します。"production"名前空間はいくらでも使える
  ようにします。

クラスタの資源量の合計が、名前空間のクオータの合計よりも少ない場合、リソースの競合が発生する
可能性があります。これは先着順の原理で取り扱われます。

競合もクオータの変更も既に作成されたリソースには影響しません。

## リソースクオータの有効化 {#enabling-resource-quota}

リソースクオータのサポートは多くのKubernetesディストリビューションでデフォルトで有効になっています。
これはAPIサーバの`--enable-admission-plugins=`フラグに`ResourceQuota`が指定されていれば
有効になります。

名前空間に`ResourceQuota`があれば、その名前空間でリソースクオータが強制されます。

## 計算リソースクオータ {#compute-resource-quota}

与えられた名前空間で要求される[計算リソース](/docs/concepts/configuration/manage-compute-resources-container/)の
総量を制限できます。

以下のリソースタイプがサポートされています。

| リソース名 | 説明 |
| --------------------- | ----------------------------------------------------------- |
| `cpu` | 非終了状態のすべてのPodに対して、CPUリクエストの合計がこの値を超えない。 |
| `limits.cpu` | 非終了状態のすべてのPodに対して、CPU制限の合計がこの値を超えない。 |
| `limits.memory` | 非終了状態のすべてのPodに対して、メモリ制限の合計がこの値を超えない。 |
| `memory` | 非終了状態のすべてのPodに対して、メモリリクエストの合計がこの値を超えない。 |
| `requests.cpu` | 非終了状態のすべてのPodに対して、CPUリクエストの合計がこの値を超えない。 |
| `requests.memory` | 非終了状態のすべてのPodに対して、メモリリクエストの合計がこの値を超えない。 |

### 拡張リソースに対するリソースクオータ {#resource-quota-for-extended-resources}

上で言及したリソースに加え、1.10では、
[拡張リソース](/docs/concepts/configuration/manage-compute-resources-container/#extended-resources)
に対するクオータサポートが追加されました。

拡張リソースに対するオーバコミットを許さないようにと、同じ拡張リソースにクオータで`requests`と`limits`を
指定しても意味がありません。拡張リソースに対しては、現在`requests.`接頭辞を持つクオータアイテムのみが
許されています。

GPUリソースを例にとると、リソース名が`nvidia.com/gpu`で、名前空間でリクエストされるGPUの総数を4に制限
したいとすると、クオータは次のように定義されます。

* `requests.nvidia.com/gpu: 4`

詳細は[クオータの閲覧と設定](#viewing-and-setting-quotas)を参照してください。

## ストレージリソースクオータ {#storage-resource-quota}

与えられた名前空間で要求される[ストレージリソース](/ja/docs/concepts/storage/persistent-volumes/)の総量を
制限できます。

加えて、関連するストレージクラスに基づくストレージリソースの使用量も制限できます。

| リソース名 | 説明 |
| --------------------- | ----------------------------------------------------------- |
| `requests.storage` | すべてのPersistentVolumeClaimに対して、ストレージリクエストの合計がこの値を超えない。 |
| `persistentvolumeclaims` | 名前空間に存在できる[PersistentVolumeClaim](/ja/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)の総数。 |
| `<storage-class-name>.storageclass.storage.k8s.io/requests.storage` | storage-class-nameに関連しているすべてのPersistentVolumeClaimに対して、ストレージリクエストの総量がこの値を超えない。 |
| `<storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims` | storage-class-nameに関連しているすべてのPersistentVolumeClaimに対して、名前空間に存在できる[PersistentVolumeClaim](/ja/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)の総数。 |

例えば、オペレータが`gold`ストレージクラスのクオータを`bronze`ストレージクラスと分けたいとすると、
オペレータはクオータを次のように定義できます。

* `gold.storageclass.storage.k8s.io/requests.storage: 500Gi`
* `bronze.storageclass.storage.k8s.io/requests.storage: 100Gi`

1.8では、ローカルの一時ストレージに対するクオータサポートがアルファ機能として追加されました。

| Resource Name | Description |
| ------------------------------- |----------------------------------------------------------- |
| `requests.ephemeral-storage` | 名前空間のすべてのPodに対して、ローカル一時ストレージリクエストの合計がこの値を超えない。 |
| `limits.ephemeral-storage` | 名前空間のすべてのPodに対して、ローカル一時ストレージリクエストの合計がこの値を超えない。 |

## オブジェクトカウントクオータ {#object-count-quota}

1.9では以下の文法を使う、すべての標準的な名前空間リソースタイプのクオータのサポートが追加されました。

* `count/<resource>.<group>`

ユーザがオブジェクトカウントクオータで制限したいリソースの例を挙げます。

* `count/persistentvolumeclaims`
* `count/services`
* `count/secrets`
* `count/configmaps`
* `count/replicationcontrollers`
* `count/deployments.apps`
* `count/replicasets.apps`
* `count/statefulsets.apps`
* `count/jobs.batch`
* `count/cronjobs.batch`
* `count/deployments.extensions`

`count/*`リソースクオータを使うと、オブジェクトがサーバストレージに存在すれば、クオータに対して変更されます。
これらのクオータのタイプは、ストレージリソースの枯渇を防ぐのに便利です。例えば、大きなサイズのSecretの数を
制限したいと考えるかもしれません。クラスタ内のSecretが多すぎると、サーバやコントローラの起動が妨げられます。
サービス拒否につながる、多くのJobを生成する不十分な構成のCronJobから守るためにJobの制限を選択することもできます。

1.9以前では、限られたリソースに対してジェネリックオブジェクトカウントクオータを実行することが可能でした。
加えて、リソースのタイプによって、さらにクオータを制限することができます。

以下のタイプがサポートされています。

| リソース名 | 説明 |
| ------------------------------- | ------------------------------------------------- |
| `configmaps` | その名前空間に存在できるConfigMapの総数 |
| `persistentvolumeclaims` | その名前空間に存在できる[PersistentVolumeClaim](/ja/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)の総数 |
| `pods` | その名前空間に存在できる、非終了状態のPodの総数。`.status.phase in (Failed, Succeeded)`が真であれば、Podは終了状態です。 |
| `replicationcontrollers` | その名前空間に存在できるReplicationコントローラの総数 |
| `resourcequotas` | その名前空間に存在できる[ResourceQuota](/ja/docs/reference/access-authn-authz/admission-controllers/#resourcequota)の総数 |
| `services` | その名前空間に存在できるServiceの総数 |
| `services.loadbalancers` | その名前空間に存在できる、LoadBalancerタイプのServiceの総数 |
| `services.nodeports` | その名前空間に存在できるNodePortタイプのServiceの総数 |
| `secrets` | その名前空間に存在できるSecretの総数 |

例えば、`pods`クオータは、終了していない作成された`pod`の数をカウントし、上限を強制します。ユーザが大量の
小さなPodを作成して、Pod IPが枯渇するのを避けるために、`pod`クオータを設定したいと思うかもしれません。

## クオータスコープ {#quota-scopes}

各クオータは関連するスコープのセットを持つことができます。クオータは、列挙されたスコープの共通部分に
マッチするリソースの使用量のみを計測します。

スコープがクオータに追加されると、そのスコープに関連するリソースの数を制限します。許可されたセットの
外側のクオータで指定されたリソースは検証エラーとなります。

| スコープ | 説明 |
| ----- | ----------- |
| `Terminating` | `spec.activeDeadlineSeconds >= 0`のPodにマッチします |
| `NotTerminating` | `spec.activeDeadlineSeconds is nil`のPodにマッチします |
| `BestEffort` | サービスがベストエフォート品質のPodにマッチします |
| `NotBestEffort` | サービスがベストエフォート品質でないPodにマッチします |

`BestEffort`スコープは`pods`リソースを追跡するクオータを制限します。

`Terminating`と`NotTerminating`、`NotBestEffort`スコープは以下のリソースを追跡するクオータを制限します。

* `cpu`
* `limits.cpu`
* `limits.memory`
* `memory`
* `pods`
* `requests.cpu`
* `requests.memory`

### PriorityClassごとのリソースクオータ {#resource-quota-per-priorityclass}

{{< feature-state for_k8s_version="1.12" state="beta" >}}

Podは指定した[優先度](/docs/concepts/configuration/pod-priority-preemption/#pod-priority)で作成することが
できます。クオータスペックの`scopeSelector`フィールドを使って、Podの優先度に応じたシステム消費量の制御が
できます。

クオータスペックの`scopeSelector`がPodを選択する場合のみ、クオータはマッチし使用されます。

{{< note >}}
**メモ:** PriorityClassごとのリソースクオータを使う前に、`ResourceQuotaScopeSelectors` Feature Gateを
有効にする必要があります。
{{< /note >}}

この例はクオータオブジェクトを生成し、指定した優先度のPodにマッチします。この例は次のように動作します。

- クラスタ内のPodは、"low", "medium", "high"の3つのPriorityClassのうちの1つを持ちます
- 各優先度に対して、1つのクオータオブジェクトが作成されます。

次のYAMLを`quota.yml`に保存します。

```yaml
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-high
  spec:
    hard:
      cpu: "1000"
      memory: 200Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["high"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-medium
  spec:
    hard:
      cpu: "10"
      memory: 20Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["medium"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-low
  spec:
    hard:
      cpu: "5"
      memory: 10Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["low"]
```

`kubectl create`を使って、YAMLを適用します。

```shell
kubectl create -f ./quota.yml
```

```shell
resourcequota/pods-high created
resourcequota/pods-medium created
resourcequota/pods-low created
```

`kubectl describe quota`を使って、`Used`のクオータが`0`であることを確認します。

```shell
kubectl describe quota
```

```shell
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     1k
memory      0     200Gi
pods        0     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

優先度が"high"のPodを作成します。以下のYAMLを`high-priority-pod.yml`に保存します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
  priorityClassName: high
```

これを`kubectl create`で適用します。

```shell
kubectl create -f ./high-priority-pod.yml
```

"high"優先度のクオータ`pods-high`の"Used"状態が変わり、ほかの2つのクオータは変わらないことを
確認します。

```shell
kubectl describe quota
```

```shell
Name:       pods-high
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         500m  1k
memory      10Gi  200Gi
pods        1     10


Name:       pods-low
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     5
memory      0     10Gi
pods        0     10


Name:       pods-medium
Namespace:  default
Resource    Used  Hard
--------    ----  ----
cpu         0     10
memory      0     20Gi
pods        0     10
```

`scopeSelector`は`operator`フィールドで以下の値をサポートします。

* `In`
* `NotIn`
* `Exist`
* `DoesNotExist`

## 要求 対 上限 {#requests-vs-limits}

計算リソースを割り当てる時、各コンテナはCPUやメモリに対して、要求や上限の値を指定できます。クオータは
これらの値を制限するように構成できます。

クオータが`requests.cpu`や`requests.memory`で指定した値を持っていれば、作成されるコンテナは
それらのリソースに対して明示的な要求が必要となります。クオータが`limits.cpu`や`limits.memory`で
指定した値を持っていれば、作成されるコンテナはそれらのリソースに対して明示的な上限が必要となります。

## クオータの閲覧と設定 {#viewing-and-setting-quotas}

kubectlはクオータの作成、更新、閲覧をサポートします。

```shell
kubectl create namespace myspace
```

```shell
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.nvidia.com/gpu: 4
EOF
```

```shell
kubectl create -f ./compute-resources.yaml --namespace=myspace
```

```shell
cat <<EOF > object-counts.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
EOF
```

```shell
kubectl create -f ./object-counts.yaml --namespace=myspace
```

```shell
kubectl get quota --namespace=myspace
```

```shell
NAME                    AGE
compute-resources       30s
object-counts           32s
```

```shell
kubectl describe quota compute-resources --namespace=myspace
```

```shell
Name:                    compute-resources
Namespace:               myspace
Resource                 Used  Hard
--------                 ----  ----
limits.cpu               0     2
limits.memory            0     2Gi
pods                     0     4
requests.cpu             0     1
requests.memory          0     1Gi
requests.nvidia.com/gpu  0     4
```

```shell
kubectl describe quota object-counts --namespace=myspace
```

```shell
Name:                   object-counts
Namespace:              myspace
Resource                Used    Hard
--------                ----    ----
configmaps              0       10
persistentvolumeclaims  0       4
replicationcontrollers  0       20
secrets                 1       10
services                0       10
services.loadbalancers  0       2
```

kubectlは`count/<resource>.<group>`という文法を使った、すべての標準的な名前空間リソースに対する
オブジェクトカウントクオータもサポートします。

```shell
kubectl create namespace myspace
```

```shell
kubectl create quota test --hard=count/deployments.extensions=2,count/replicasets.extensions=4,count/pods=3,count/secrets=4 --namespace=myspace
```

```shell
kubectl run nginx --image=nginx --replicas=2 --namespace=myspace
```

```shell
kubectl describe quota --namespace=myspace
```

```shell
Name:                         test
Namespace:                    myspace
Resource                      Used  Hard
--------                      ----  ----
count/deployments.extensions  1     2
count/pods                    2     3
count/replicasets.extensions  1     4
count/secrets                 1     4
```

## クオータとクラスタ容量 {#quota-and-cluster-capacity}

`ResourceQuota`はクラスタ容量とは独立しています。これらは絶対的な単位で表現されます。したがって、
クラスタにノードを追加しても、自動的に各名前空間へリソースを追加したりは *しません* 。

時々、次のような複雑なポリシが要求されることがあります。

  - いくつかのチームでクラスタリソース全体を相対的に分割する
  - 各テナントが必要に応じてリソース使用量を増やせるようにするが、予期せぬリソース枯渇を防ぐために
    寛大な制限が設けられる
  - 名前空間からの要求を検知し、ノードを追加し、クオータを増やす

このようなポリシは、クオータの使用量を監視し、ほかのシグナルに応じて各名前空間のハードリミットを
調整する"コントローラ"を書くことで、`ResourceQuotas`を基礎的な要素として使って実装できます。

リソースクオータはクラスタリソースの集合を分割しますが、ノードの制約は作らないことに注意してください。
いくつかの名前空間のPodが同じノードで実行されることもあります。

## デフォルトでPriorityClassの使用量を制限する {#limit-priority-class-consumption-by-default}

特定の優先度 (例えば"cluster-services") のPodを、マッチするクオータオブジェクトが存在する名前空間でだけ
許可することが要求されることがあります。

この仕組みでは、オペレータは、ある高いPriorityClassの使用量を、これらのPriorityClassをデフォルトで使う
限定された数に名前空間に制限することができます。

これを使うためには、kube-apiserverのフラグ `--admission-control-config-file`に以下の構成ファイルを
渡します。

```yaml
apiVersion: apiserver.k8s.io/v1alpha1
kind: AdmissionConfiguration
plugins:
- name: "ResourceQuota"
  configuration:
    apiVersion: resourcequota.admission.k8s.io/v1beta1
    kind: Configuration
    limitedResources:
    - resource: pods
    matchScopes:
    - operator : In
      scopeName: PriorityClass
      values: ["cluster-services"]
```

これで、"cluster-services"のPodは、`scopeSelector`がマッチするクオータオブジェクトが存在する名前空間のみを
許可されるようになります。

例えば、
```yaml
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["cluster-services"]
```

詳細は[LimitedResources](https://github.com/kubernetes/kubernetes/pull/36765)と
[Quota support for priority class design doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/pod-priority-resourcequota.md)
を参照してください。

## 例 {#example}

[ResourceQuotaの使い方についての詳細な例](/docs/tasks/administer-cluster/quota-api-object/)を参照してください。

{{% /capture %}}

{{% capture whatsnext %}}

詳細は[ResourceQuotaの設計ドキュメント](https://git.k8s.io/community/contributors/design-proposals/resource-management/admission_control_resource_quota.md)を参照してください。

{{% /capture %}}
