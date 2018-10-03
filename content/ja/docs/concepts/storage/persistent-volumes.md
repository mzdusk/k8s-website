---
title: Persistent Volumes
content_template: templates/concept
weight: 20
---

{{% capture overview %}}

この文書では Kubernetes の `PersistentVolume` の現状について記述します。[Volume](/ja/docs/concepts/storage/volumes/) に精通していることを勧めます。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## 導入

ストレージを管理することは、計算資源を管理することとは分離した問題です。`PersistentVolume` サブシステムはユーザと管理者にどのようなストレージが提供されていて、それをどのように使うのかの詳細を抽象化する API を提供します。ここでは 2 つの新しい API リソースである、`PersistentVolume` と `PersistentVolumeClaim` を紹介します。

`PersistentVolume` (PV) は管理者によって提供されているクラスター内のストレージの一部です。ノードがクラスターリソースであるように、これもクラスターのリソースです。PV は Volume のようなボリュームプラグインですが、PV を使う個々の Pod とは独立したライフサイクルを持っています。この API オブジェクトは NFS や iSCSI、クラウドプロバイダ指定のストレージシステムといったストレージの実装の詳細をキャプチャーします。

`PersistentVolumeClaim` (PVC) はユーザによるストレージの要求です。これは Pod と似ています。Pod はノードリソースを使いますが、PVC は PV リソースを使います。Pod はリソース水準 (CPU とメモリ) を指定して要求できます。Claim はサイズとアクセスモード (たとえば、一度だけマウントできる読み書きアクセスや何度もマウントできる読み込みアクセスなど) を指定して要求できます。

`PersistentVolumeClaim` はユーザに抽象化ストレージリソースを使うことを許可しますが、ユーザは異なる問題に対して、パフォーマンスのようなさまざまなプロパティを指定する `PersistentVolume` を必要とするのが一般的です。クラスター管理者はボリューム実装の詳細をユーザに見せることなく、サイズやアクセスモードだけでないさまざまな種類の `PersistentVolume` を提供できる必要があります。これらのニーズに対して `StorageClass` リソースがあります。

[例を実行する詳細なウォークスルー](/ja/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)を参照してください。

## Volume と Claim のライフサイクル

PV はクラスターのリソースです。PVC はこれらのリソースへの要求で、リソースをチェックする Claim としても振る舞います。PV と PVC 間のやりとりは以下のライフサイクルに従います。

### プロビジョニング {#provisioning}

PV は静的と動的の2つの方法で供給されます。

#### 静的

クラスター管理者は多くの PV を作成します。これらはクラスタユーザが利用可能な実ストレージの詳細を持ちます。これらは Kubernetes API に存在しており、利用可能です。

#### 動的

ユーザの `PersistentVolumeClaim` にマッチする管理者が作成した静的 PV が無ければ、クラスターは PVC のために特別に動的供給を試みることができます。この供給は `StorageClass` に基づきます。PVC は [Storage Class](/ja/docs/concepts/storage/storage-classes/) を要求しなければならず、管理者は動的供給を実行するためのクラスを作成、構成しておかなければなりません。クラス `""` を要求する Claim は事実上、動的供給を無効化します。

Storage Class に基づいた動的ストレージ供給を有効にするには、クラスタ管理者が API サーバで `DefaultStorageClass` [アドミッションコントローラ](/ja/docs/admin/admission-controllers/#defaultstorageclass) を有効にする必要があります。これは例えば、`DefaultStorageClass` が API サーバコンポネントの `--enable-admission-plugins` フラグのカンマ区切りの順序付きリストに確実に含まれるようにすることで実現できます。API サーバコマンドラインフラグについての詳細は、[kube-apiserver](/ja/docs/admin/kube-apiserver/) のドキュメントを参照してください。

### 結合

ユーザは必要とするストレージ容量とアクセスモードを指定した `PersistentVolumeClaim` を作成します (動的供給の場合は既に作成されています)。マスタの制御ループは新しい PVC を監視し、(可能ならば) マッチする PV を見つけ、それらを結合します。もし PV が新しい PVC に対して動的に供給されれば、ループは常に PV を PVC に結合します。さもなければ、ユーザは少なくとも要求した結果は得られますが、ボリュームを超過する可能性があります。一度結合されると、`PersistentVolumeClaim` 結合はどのように結合されたかにかかわらず排他的です。PVC と PV の結合は1対1マッピングです。

マッチするボリュームが存在しなければ Claim はいつまでも結合されないままです。Claim はマッチするボリュームが利用可能になれば結合されます。例えば、多くの 50Gi PV を供給するクラスターは 100Gi を要求する PVC にマッチしません。この PVC は 100Gi の PV がクラスタに追加されれば結合されます。

### 使用

Pod は Claim をボリュームとして使います。クラスターは結合されたボリュームを探すために Claim を検査し、そのボリュームを Pod にマウントします。複数のアクセスモードをサポートするボリュームに対して、ユーザは Claim を Pod のボリュームとして使う時にどのモードを使うのかを指定します。

一度ユーザが Claim を持ち、その Claim が結合されれば、結合された PV は必要なだけユーザに属する。ユーザは Pod をスケジュールし、Pod のボリュームブロックに `persistentVolumeClaim` を含めることで請求した PV にアクセスする。[詳細な文法は下記を参照](#volume-としての-claim)。

### Storage Object in Use Protection

Storage Object in Use Protection の目的はPodによって使用中の Persistent Volume Claim (PVC) と PVC に結合している Persistent Volume (PV) がファイルシステムから削除されないことを保証することです。

{{< note >}}
**メモ** PVC は Pod の状態が `Pending` でノードに割り当てられているか、Pod の状態が `Running` であれば使用中です。
{{< /note >}}

[Storage Object in Use Protection機能](/ja/docs/tasks/administer-cluster/storage-object-in-use-protection/)が有効であるとき、ユーザが Pod によって使用中の PVC を削除しても、その PVC はすぐには削除されません。PVC の削除は、その PVC がどの Pod からも使われなくなるまで延期され、管理者が PVC に結合された PV を削除しても、PV はすぐには削除されません。PV の削除はその PV が PVC と結合されなくなるまで延期されます。

PVC の状態が `Terminating` で `Finalizers` リストに `kubernetes.io/pvc-protection` が含まれていると PVC が保護されていることがわかります。

```shell
kubectl describe pvc hostpath
Name:          hostpath
Namespace:     default
StorageClass:  example-hostpath
Status:        Terminating
Volume:        
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-class=example-hostpath
               volume.beta.kubernetes.io/storage-provisioner=example.com/hostpath
Finalizers:    [kubernetes.io/pvc-protection]
...
```

PV の状態が `Terminating` で `Finalizers` リストに `kubernetes.io/pv-protection` が含まれていると PV が保護されていることがわかります。

```shell
kubectl describe pv task-pv-volume
Name:            task-pv-volume
Labels:          type=local
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    standard
Status:          Available
Claim:           
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        1Gi
Message:         
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /tmp/data
    HostPathType:  
Events:            <none>
```

### 再請求

ユーザがボリュームを使い終えると、リソースの再請求を可能にするように PVC オブジェクトを API から削除できます。`PersistentVolume` に対する再請求ポリシーはクラスターにボリュームが Claim からリリースされた後に何をするかを伝えます。現在、Retain、Recycle、Deleteを指定できます。

#### Retain

`Retain` 再請求ポリシーは手動でのリソースの再請求を許可します。`PersistentVolumeClaim` が削除されると、`PersistentVolume` は存在しつづけ、"リリースされた"とみなされます。しかし、前の利用者のデータがボリューム上に残っているため、ほかの Claim に対してはまだ利用できません。管理者は次のステップにより手動でボリュームを再請求できます。

1. `PersistentVolume` を削除します。(AWS EBSやGCE PD、Azure Disk、Cinderボリュームなどの) 外部インフラにある関連するストレージ資産はPVが削除された後も残ります。
2. 適宜、関連するストレージ資産のデータを手動でクリーンアップします。
3. 手動で関連するストレージ資産を削除するか、同じストレージ資産を再利用するのであれば、新しい `PersistentVolume` を作成します。

#### Delete

`Delete` 再請求ポリシーをサポートするボリュームプラグインに対して、削除は外部インフラのストレージ資産と同時に、Kubernetes から `PersistentVolume` も削除します。動的供給されたボリュームは[その `StorageClass` の再請求ポリシ](#再請求ポリシ)を継承し、デフォルトは `Delete` です。管理者はユーザの期待に応じて `StorageClass` を構成すべきであり、そうでなければ PV は作成された後で編集されなければなりません。[PersistentVolume の Reclaim Policy の変更](/ja/docs/tasks/administer-cluster/change-pv-reclaim-policy/)を参照してください。

#### Recycle

{{< warning >}}
**警告** `Recycle` 再請求ポリシは非推奨です。かわりの推奨されるアプローチは動的供給を使うことです。
{{< /warning >}}

ボリュームプラグインでサポートされていれば、`Recycle` 再請求ポリシーは基本的な削除を行い (`rm -rf /thevolume/*`)、新しい Claim に対して再び利用可能にします。

しかしながら、管理者は [ここ](/ja/docs/admin/kube-controller-manager/)で述べられたような Kubernetes コントローラマネージャコマンドライン引数を使って、カスタムリサイクル Pod テンプレートを構成できます。カスタムリサイクル Pod テンプレートは以下の例のように volume 仕様を含めなければなりません。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    hostPath:
      path: /any/path/it/will/be/replaced
  containers:
  - name: pv-recycler
    image: "k8s.gcr.io/busybox"
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```

カスタムリサイクル Pod テンプレートの volumes パートで指定された特定のパスは、リサイクルされるボリュームの特定のパスに置き換えられます。

### Persistent Volume Claim の拡張

{{< feature-state for_k8s_version="v1.8" state="alpha" >}}
{{< feature-state for_k8s_version="v1.11" state="beta" >}}

PersistentVolumeClaim (PVC) の拡張のサポートはデフォルトで有効です。以下のタイプのボリュームを拡張できます。

* gcePersistentDisk
* awsElastivBlockStore
* Cinder
* glusterfs
* rbd
* Azure File
* Azure Disk
* Portworx

Storage Class の `allowVolumeExpansion` フィールドが true に設定されていれば PVC の拡張のみができる。

``` yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gluster-vol-default
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://192.168.10.100:8080"
  restuser: ""
  secretNamespace: ""
  secretName: ""
allowVolumeExpansion: true
```

PVC に対してより大きなボリュームを要求するため、その PVC オブジェクトを編集し、より大きなサイズを指定します。これは `PersistentVolume` の裏にあるボリュームを拡張するためのトリガです。新しい `PersistentVolume` がその Claim を満たすために作成されることは決してありません。かわりに、既存のボリュームがリサイズされます。

#### ファイルシステムを含むボリュームのリサイズ

ファイルシステムが XFS、Ext3、Ext4 であれば、ファイルシステムを含むボリュームのリサイズができます。

ボリュームがファイルシステムを含んでいると、そのファイルシステムは新しい Pod が ReadWrite モードの `PersistentVolumeClaim` を使って起動される時にリサイズされます。したがって、Pod または Deployment がボリュームを使い、それを広げたいのであれば、controller-manager のクラウドプロバイダによってボリュームが拡張された後でPodを削除または再作成する必要があります。`kubectl describe pvc` コマンドを実行することでリサイズ操作の状態をチェックできます。

```
kubectl describe pvc <pvc_name>
```

`PersistentVolumeClaim` の状態が `FileSystemResizePending` であれば、その PersistentVolumeClaim を使う Pod の再作成は安全です。

#### 使用中の PersistentVolumeClaim のリサイズ

{{< feature-state for_k8s_version="v1.11" state="alpha" >}}

使用中の PVC の拡張はアルファ機能です。使うためには、`ExpandInUsePersistentVolumes` feature gate を有効にします。この場合、拡張する PVC を使う Pod や Deployment を削除または再作成する必要はありません。あらゆる使用中の PVC はファイルシステムが拡張されるとすぐに使用可能になります。この機能は Pod または Deployment によって使われていない PVC には効果がありません。拡張を完了する前にその PVC を使う Pod を作成する必要があります。

{{< note >}}
**メモ** EBS ボリュームの拡張は時間のかかる操作です。また、変更は 6 時間に 1 回のボリューム別クオータがあります。
{{< /note >}}

## PersistentVolume のタイプ

`PersistentVolume` タイプはプラグインとして実装されます。Kubernetes は現在以下のプラグインをサポートします。

* GCEPersistentDisk
* AWSElasticBlockStore
* AzureFile
* AzureDisk
* FC (Fibre Channel)
* FlexVolume
* Flocker
* NFS
* iSCSI
* RBD (Ceph Block Device)
* CephFS
* Cinder (OpenStack block storage)
* Glusterfs
* VsphereVolume
* Quobyte Volumes
* HostPath (単一ノードでのテストのみ -- どのような方法でもローカルストレージはサポートされず、マルチノードクラスタでは利用できません)
* Portworx Volumes
* ScaleIO Volumes
* StorageOS

## Persistent Volume

各PVコンテナはスペックと状態を含み、これはボリュームの仕様と状態です。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

### Capacity

通常、PV は特定のストレージ容量を持ちます。これは PV の `capacity` 属性を使って指定されます。`capacity` で指定する単位を理解するために、Kubernetes の [Resource Model](https://git.k8s.io/community/contributors/design-proposals/scheduling/resources.md) を参照してください。

現在、ストレージサイズのみが設定もしくは要求できるリソースです。将来的にはIOPSやスループットを指定できるかもしれません。

### Volume Mode

{{< feature-state for_k8s_version="v1.9" state="alpha" >}}

この機能を有効にするためには、apiserver、コントローラマネージャ、kubeletで `BlockVolume` feature gateを有効にします。

Kubernetes 1.9以前では、すべてのボリュームプラグインが Persistent Volume 上にファイルシステムを作っていました。現在は raw ブロックデバイスを使うために `volumeMode` を `raw` に、もしくはファイルシステムを使うために `filesystem` に設定することができます。値が省略された場合は `filesystem`がデフォルトで使われます。これは任意のAPIパラメータです。

### Access Mode

`PersistentVolume` はリソースプロバイダによってサポートされる方法でホストにマウントされます。以下の表で示すように、プロバイダは異なる能力を持っており、各 PV のアクセスモードは特定のボリュームによってサポートされる指定のモードに設定されます。例えば、NFS は複数の読み書きクライアントをサポートできますが、特定の NFS PV はリードオンリーでサーバに提供されるかもしれません。各 PV は指定した PV の能力を示す自身のアクセスモードの集合を得ます。

以下のアクセスモードがあります。

* ReadWriteOnce: ボリュームは単一ノードから読み書き可能でマウントできます
* ReadOnlyMany: ボリュームは複数のノードからリードオンリーでマウントできます
* ReadWriteMany: ボリュームは複数のノードから読み書き可能でマウントできます

CLIでは以下のように略されます。

* RWO: ReadWriteOnce
* ROX: ReadOnlyMany
* RWX: ReadWriteMany

> __重要！__ ボリュームは、複数サポートしていたとしても、一度に1つのアクセスモードのみを使ってマウントされます。例えば、GCEPersistentDisk は単一ノードでの ReadWriteOnce か複数ノードでの ReadOnlyMany でマウントできますが、同時には使えません。

| Volume Plugin        | ReadWriteOnce| ReadOnlyMany| ReadWriteMany|
| :---                 |     :---:    |    :---:    |    :---:     |
| AWSElasticBlockStore | &#x2713;     | -           | -            |
| AzureFile            | &#x2713;     | &#x2713;    | &#x2713;     |
| AzureDisk            | &#x2713;     | -           | -            |
| CephFS               | &#x2713;     | &#x2713;    | &#x2713;     |
| Cinder               | &#x2713;     | -           | -            |
| FC                   | &#x2713;     | &#x2713;    | -            |
| FlexVolume           | &#x2713;     | &#x2713;    | -            |
| Flocker              | &#x2713;     | -           | -            |
| GCEPersistentDisk    | &#x2713;     | &#x2713;    | -            |
| Glusterfs            | &#x2713;     | &#x2713;    | &#x2713;     |
| HostPath             | &#x2713;     | -           | -            |
| iSCSI                | &#x2713;     | &#x2713;    | -            |
| Quobyte              | &#x2713;     | &#x2713;    | &#x2713;     |
| NFS                  | &#x2713;     | &#x2713;    | &#x2713;     |
| RBD                  | &#x2713;     | &#x2713;    | -            |
| VsphereVolume        | &#x2713;     | -           | - (Podが共同配置されていれば動作)  |
| PortworxVolume       | &#x2713;     | -           | &#x2713;     |
| ScaleIO              | &#x2713;     | &#x2713;    | -            |
| StorageOS            | &#x2713;     | -           | -            |

### クラス

PV はクラスを持つことができ、これは `storageClassName` 属性に [StorageClass](/ja/docs/concepts/storage/storage-classes/) の名前を設定することで指定されます。特定のクラスのPVはそのクラスを要求するPVCにのみ結合できます。`storageClassName` のない PV はクラスを持たず、特定のクラスを要求しない PVC にのみ結合できます。

### 再請求ポリシ

現在の再請求ポリシーは以下です。

* Retain: 手動での再請求
* Recycle: 基本的な削除 (`rm -rf /thevolume/*`)
* Delete: AWS EBS, GCE PD, Azure Disk, OpenStack Cinder volumeのような関連するストレージ資産は削除されます

現在、NFS と HostPath でのみ Recycle をサポートします。AWS EBS, GCE PD, Azure Disk, Cinder volume は Delete をサポートします。

### マウントオプション

Kubernetes 管理者は Persistent Volume がノードにマウントされる時に使う追加のマウントオプションを指定できます。

以下のボリュームタイプでマウントオプションをサポートします。

* GCEPersistentDisk
* AWSElasticBlockStore
* AzureFile
* AzureDisk
* NFS
* iSCSI
* RBD (Ceph Block Device)
* CephFS
* Cinder (OpenStack block storage)
* Glusterfs
* VsphereVolume
* Quotebyte Volumes

マウントオプションは検証されないので、不正であれば単純にマウントに失敗します。

### フェーズ

ボリュームは以下のフェーズの1つになります。

* Available: Claimにまだ結合していない空きリソース
* Bound: Claimに結合済みのボリューム
* Released: Claimは削除されたが、リソースはまだクラスターによって再請求されていない
* Failed: ボリュームは自動再請求に失敗した

CLI では PV に結合している PVC の名前が表示される。

## PersistentVolumeClaim {#persistentvolumeclaims}

各 PVC にはスペックとステータスが含まれ、これは Claim の仕様とステータスです。

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

### Access Mode

Claim は指定したアクセスモードでストレージを要求すると、同じアクセスモードのボリュームを使います。

### Volume Mode

Claim はファイルシステムかブロックデバイスのどちらかのボリュームの使用を指示するために同じボリュームモードのボリュームを使います。

### リソース

Claim は Pod のように、リソース量を指定できます。この場合、要求はストレージに対してです。同じ[リソースモデル](https://git.k8s.io/community/contributors/design-proposals/scheduling/resources.md)がボリュームと Claim の両方に適用されます。

### セレクタ

Claim はボリュームの集合をさらにフィルタするためにラベルセレクタを指定できます。ラベルがセレクタにマッチするボリュームだけを Claim と結合させることができます。セレクタは2つのフィールドで構成できます。

* `matchLabels`: ボリュームはこの値を持たなければなりません
* `matchExpressions`: キーや値のリスト、キーと値に関連する演算子を指定することで作られる必要条件のリスト。有効な演算子はIn, NotIn, Exists, DoesNotExistです。

`matchLabels` と `matchExpressions` の両方の必要条件はANDで結合されます。マッチするためにはすべてを満たさなければなりません。

### クラス

Claim は `storageClassName` 属性を使って[StorageClass](/ja/docs/concepts/storage/storage-classes/) の名前を指定することで特定のクラスを要求できます。要求されたクラスの PV のみが PVC に結合できます。

PVC はクラスを要求する必要はありません。`storageClassName` を `""` に設定した PVC はクラスのない PV を要求するので、クラスのない PV にしか結合しません。`storageClassName` のない PVC はこれとは異なり、クラスターで [`DefaultStorageClass` アドミッションプラグイン](/docs/admin/admission-controllers/#defaultstorageclass) が有効かどうかで扱いが変わります。

* アドミッションプラグインが有効であれば、管理者はデフォルトの `StorageClass` を指定できます。`storageClassName` を持たないすべての PVC はそのデフォルトの PV にのみ結合できます。デフォルト `StorageClass` の指定は `StorageClass` オブジェクトのアノテーション `storageclass.kubernetes.io/is-default-class` を"true"に設定することで行われます。管理者がデフォルトを指定していなければ、クラスターはアドミッションプラグインが無効であるような形で PVC の作成に対応します。複数のデフォルトが指定されていれば、アドミッションプラグインはすべてのPVCの作成を禁止します。
* アドミッションプラグインが無効であれば、デフォルトの `StorageClass` は存在しません。`storageClassName`を持たないすべての PVC はクラスを持たない PV にのみ結合できます。この場合、`storageClassName` を持たない PVC は `storageClassName` を `""` に設定したPVCと同じように扱われます。

インストール方法によって、デフォルト StorageClass はインストール中にアドオンマネージャによりKubernetesクラスタにデプロイされます。

PVC で `StorageClass` の要求に加えて `selector` を指定すると、必要条件は AND で結合されます。要求されたクラスとラベルの両方を持つ PV のみがその PVC に結合できます。

{{< note >}}
**メモ:** 現在、空でない `selector` を持つ PVC は動的に供給された PV を持つことができません。
{{< /note >}}

## Volume としての Claim

Pod はボリュームとして Claimを 使うことでストレージにアクセスします。Claim はその Claim を使う Pod と同じ名前空間に存在しなければなりません。クラスタは Pod の名前空間にある Claim を探し、Claim の裏にある `PersistentVolume` を得るために使います。その後、Volume はホストにマウントされPodに渡ります。

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: dockerfile/nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```


### 名前空間についてのメモ

`PersistentVolume` 結合は排他的で、`PersistentVolumeClaim` は名前空間オブジェクトなので、Manyモード (`ROX`, `RWX`) のClaimへのマウントは名前空間につき1つのみ可能です。

## Raw ブロックボリュームのサポート

{{< feature-state for_k8s_version="v1.9" state="alpha" >}}

Rawブロックボリュームのサポートを有効にするには、apiserver, controller-manager, kubeletで `BlockVolume` feature gate を有効にします。

以下のボリュームプラグインで、適用可能な場所での動的供給を含む、Rawブロックボリュームをサポートします。

* AWSElasticBlockStore
* AzureDisk
* FC (Fibre Channel)
* GCEPersistentDisk
* iSCSI
* Local volume
* RBD (Ceph Block Device)

{{< note >}}
**メモ**: Kubernetes 1.9では FC と iSCSI のみが Raw ブロックデバイスをサポートします。追加のプラグインに対するサポートは 1.10 で追加されました。
{{< /note >}}

### Raw ブロックデバイスを使う PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```

### Raw ブロックボリュームを要求する PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
```

### コンテナで Raw ブロックデバイスパスを追加する Pod スペック

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```

{{< note >}}
**メモ:** Pod に対して Raw ブロックデバイスを追加する時には、mount path のかわりに device path を指定します。
{{< /note >}}

### ブロックボリュームの結合

ユーザが `PersistentVolumeClaim` スペックの `volumeMode` フィールドを使ってRawブロックボリュームを要求すると、結合ルールはこのモードを考慮しない以前のリリースとは若干異なります。ユーザと管理者が指定する可能性のある組み合わせを列挙しました。

| PV volumeMode | PVC volumeMode  | Result           |
| --------------|:---------------:| ----------------:|
|   unspecified | unspecified     | BIND             |
|   unspecified | Block           | NO BIND          |
|   unspecified | Filesystem      | BIND             |
|   Block       | unspecified     | NO BIND          |
|   Block       | Block           | BIND             |
|   Block       | Filesystem      | NO BIND          |
|   Filesystem  | Filesystem      | BIND             |
|   Filesystem  | Block           | NO BIND          |
|   Filesystem  | unspecified     | BIND             |

{{< note >}}
**メモ:** 静的に供給されたボリュームはアルファリリースに対してのみサポートされます。管理者は Raw ブロックデバイスを利用する時にはこれらの値への配慮を行うべきです。
{{< /note >}}

## ポータブルな構成を書く

永続ストレージが必要で幅広いクラスターで実行する構成テンプレートまたは例を書いているなら、以下のパターンを使うことをお勧めします。

- (Deployment や ConfigMap を一緒にした) 構成バンドルに PersistentVolumeClaim オブジェクトを含める
- 構成を使うユーザが PersistentVolume を作成する権限を持たない可能性があるので、構成に PersistentVolume オブジェクトは含めない
- テンプレートを使う時に、ユーザに供給するストレージクラス名のオプションを与える
 - ユーザがストレージクラス名を提供すれば、その値を `persistentVolumeClaim.storageClassName` フィールドに設定する。これにより、クラスターが管理者によって有効にされたStorageClassを持っていれば、正しいストレージクラスにマッチするPVCとなる。
 - ユーザがストレージクラス名を提供しなければ、`persistentVolumeClaim.storageClassName` フィールドはnilのままにする。
  - これによりクラスターのデフォルトStorageClassを持つユーザに自動的に供給されるPVとなる。多くのクラスタ環境はデフォルトStorageClassがインストールされるか、管理者が自身のデフォルトStorageClassを作成できる。
- ツールの中で、いくらか時間がたっても結合されないPVCを監視し、ユーザに知らせる。これはクラスタが動的ストレージサポートを持たない (この場合、ユーザはマッチするPVを作るべきである) か、クラスタがストレージシステムを持たない (この場合、ユーザは要求するPVCの構成をデプロイできない) かである。

{{% /capture %}}
