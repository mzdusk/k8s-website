---
title: Volume
content_template: templates/concept
weight: 10
---

{{% capture overview %}}

コンテナのディスク上のファイルは一時的なものであり、これがコンテナで実行される重要なアプリケーションにとって問題となることがあります。まず、コンテナがクラッシュすると、kubelet はそれを再起動させるが、ファイルは失われます --- コンテナはクリーンな状態で起動されます。次に、コンテナを `Pod` 内で一緒に起動する場合、これらのコンテナ間でたびたびファイルを共有する必要があります。Kubernetes の `Volume` 抽象化はこれらの問題を両方とも解決します。

[Pod](/ja/docs/user-guide/pods) について精通していることをお勧めします。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## 背景 {#background}

Docker も [volumes](https://docs.docker.com/engine/admin/volumes/) の概念を持ちますが、いくぶんゆるく管理されます。Docker では、volume は単純にディスク上のディレクトリか他のコンテナです。生存時間は管理されず、つい最近までローカルディスクの volume があるだけでした。現在は Docker は volume ドライバーを提供していますが、今でもその機能は非常に制限されています (たとえば、Docker 1.7 ではコンテナに対して 1 つの volume ドライバーしか許されず、volume にパラメータを渡す方法がありません)。

他方で、Kubernetes の Volume は明示的な生存期間を持ちます --- Volume を含む Pod と同じです。その結果、Volume は Pod 内で実行するコンテナよりも長生きで、データはコンテナの再起動を経ても保持されます。もちろん、Podが存在しなくなれば、Volume も存在しなくなります。おそらくこれより重要なのは、Kubernetes が多くのタイプの Volume をサポートし、Pod はこれらを同時に使えるということです。

根本的には、Volume は単なるディレクトリで、何らかのデータが中にある可能性があり、Pod のコンテナがアクセス可能です。そのディレクトリがどのようにしてやって来るのか、支えとなるメディアやその内容は使用する Volume タイプによって決められます。

Volume を使うためには、Pod は使用する Volume を指定し (`.spec.volumes` フィールド)、それらのコンテナへのマウント先を指定します (`.spec.containers.volumeMounts` フィールド)。

コンテナのプロセスは [Docker イメージ](https://docs.docker.com/userguide/dockerimages/)と Volume で構成されたファイルシステムビューを見ます。Docker イメージはファイルシステム階層のルートで、volume はイメージ内の指定されたパスにマウントされます。Volume は他の Volume 上にマウントできず、他の Volume へのハードリンクも持てません。Pod 内の各コンテナは各 Volume のマウント先を独立して指定しなければなりません。

## Volume のタイプ {#types-of-volumes}

Kubernetesは幾つかのタイプのVolumeをサポートします。

* `awsElasticBlockStore`
* `azureDisk`
* `azureFile`
* `cephfs`
* `configMap`
* `csi`
* `downwardAPI`
* `emptyDir`
* `fc` (fibre channel)
* `flocker`
* `gcePersistentDisk`
* `gitRepo` (deplicated)
* `glusterfs`
* `hostPath`
* `iscsi`
* `local`
* `nfs`
* `persistentVolumeClaim`
* `projected`
* `portworxVolume`
* `quobyte`
* `rbd`
* `scaleIO`
* `secret`
* `storageos`
* `vsphereVolume`

追加の貢献を歓迎します。

### awsElasticBlockStore {#awselasticblockstore}

`awsElastivBlockStore` VolumeはAmazon Web Services (AWS) の[EBS Volume](http://aws.amazon.com/ebs/)をPodにマウントします。Podが削除された場合に削除される`emptyDir`とは異なり、EBS Volumeの内容は保持され、Volumeはただアンマウントされるだけです。これは、EBS Volumeにデータの前投入ができ、そのデータをPod間で「手渡す」ことができることを意味しています。

{{< caution >}}
**重要:** 使う前に、`aws ec2 create-volume`またはAWS APIを使ってEBS Volumeを作成しておかなければなりません。
{{< /caution >}}

`awsElasticBlockStore`を使う際には、いくつかの制限があります。

* Podを実行しているノードはAWS EC2インスタンスでなければなりません
* これらのインスタンスは、EBS Volumeと同じリージョンとアベイラビリティゾーンにある必要があります。
* EBSは単一のEC2インスタンスでのマウントのみをサポートします。

#### EBS Volumeの作成 {#creating-an-ebs-volume}

PodでEBS Volumeを使う前に、作成する必要があります。

```shell
aws ec2 create-volume --availability-zone=eu-west-1a --size=10 --volume-type=gp2
```

ゾーンがクラスタを展開しているゾーンとマッチしていることを確認してください。(また、EBS Volumeのサイズとタイプが、利用目的に合っているかもチェックしてください!)

#### AWS EBS の例 {#aws-ebs-example-configuration}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-ebs
      name: test-volume
  volumes:
  - name: test-volume
    # This AWS EBS volume must already exist.
    awsElasticBlockStore:
      volumeID: <volume-id>
      fsType: ext4
```

### azureDisk {#azuredisk}

`azureDisk`はMicrosoft Azure [Data Disk](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-about-disks-vhds/)をPodにマウントするために使います。

詳細は[ここ](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/azure_disk/README.md)を参照してください。

### azureFile {#azureFile}

`azureFile`はMicrosoft Azure File Volume (SMB 2.1と3.0) をマウントするために使います。

詳細は[ここ](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/azure_file/README.md)を参照してください。

### cephfs {#cephfs}

`cephfs` Volumeは既存のCephFS VolumeをPodにマウントできます。Podが削除された時に消える`emptyDir`とは異なり、`cephfs` Volumeの内容は保持され、Volumeはただアンマウントされるだけです。これはCephFSにデータを前投入し、Pod間でそのデータを「手渡し」できることを意味します。CephFSは同時に複数のライタにマウントできます。

{{< caution >}}
**重要:** 使う前に、共有がエクスポートされているCephサーバが必要です。
{{< /caution >}}

詳細は[CephFS example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/cephfs/)を参照してください。

### configMap {#configMap}

[`configMap`](/docs/tasks/configure-pod-container/configure-pod-configmap/)リソースは、Podの構成データを注入する手段を提供します。`ConfigMap`オブジェクトに格納されたデータは`configMap`タイブのVolumeで参照でき、Podで稼動するコンテナ化されたアプリケーションによって使われます。

`configMap`を参照する場合、参照するVolumeにその名前を単純に提供します。ConfigMapの特定のエントリに対して使うパスもカスタマイズできます。例えば、`configmap-pod`というPodに`log-config` ConfigMapをマウントするためには、次のYAMLを使います。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

`log-config` ConfigMapはVolumeとしてマウントされ、`log_level`エントリに格納されているすべての内容は、Podの"`/etc/config/log_level`"にマウントされます。このパスはVolumeの`mountPath`と`log_level`という`path`から来ていることに注意してください。

{{< caution >}}
**重要:** [ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/)は使う前に作成しておかなければなりません。
{{< /caution >}}

{{< note >}}
**メモ:** [subPath](#using-subpath) VolumeマウントとしてConfigMapを使うコンテナは、ConfigMapの更新を受け取りません。
{{< /note >}}

### downwardAPI {#downwardapi}

`downwardAPI` Volumeはアプリケーションで利用可能なDownward APIデータを作るために使われます。これはディレクトリをマウントし、リクエストされたデータをプレーンテキストファイルに書き出します。

{{< note >}}
**メモ:** [subPath](#using-subpath) VolumeマウントしてDownward APIを使うコンテナは、Downward APIの更新を受け取りません。
{{< /note >}}

詳細は[`downwardAPI` volume example](/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)を参照してください。

### emptyDir {#emptydir}

`emptyDir` VolumeはPodがノードに割り当てられた時に、最初に作られ、Podがそのノードで稼動している間だけ存在します。名前が示すように、最初は空です。Podのコンテナは、`emptyDir`を各コンテナで同じパスにも異なるパスにもマウントできますが、このVolumeにある同じファイルをすべて読み書きできます。どのような理由であれ、Podがノードから削除されると、`emptyDir`のデータは永遠に失われます。

{{< note >}}
**メモ:** コンテナのクラッシュはノードからPodを削除 *しない* ので、コンテナのクラッシュで`emptyDir`のデータは失われません。
{{< /note >}}

`emptyDir`の使用例をいくつか紹介します。

* ディスクベースのマージソートのような、スクラッチスペース
* クラッシュからのリカバリに対する長い計算のチェックポイント保存
* Webサーバコンテナがデータを提供する間、コンテンツ管理コンテナが取得するファイルの保持

デフォルトでは、`emptyDir` Volumeは、ノードが提供するメディア (ディスクやSSD、ネットワークストレージなど、環境による) のどこにでも格納されます。しかしながら、tmpfs (RAMベースのファイルシステム) にマウントするようKubernetesに伝えるため、`emptyDir.medium`フィールドに`"Memory"`を指定できます。tmpfsは非常に高速ですが、ディスクと違い、ノードの再起動で削除され、書き込んだファイルはコンテナのメモリ制限にカウントされます。

#### 例 {#example-pod}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### fc (ファイバーチャネル) {#fc}

`fc` Volumeは既存のファイバーチャネルボリュームをPodにマウントさせることができます。Volume構成の`targetWWNs`パラメータを使って、1つまたは複数のWorld Wide Nameを指定できます。複数のWWNが指定されれば、targetWWNsはこれらのWWNがマルチパス接続であると期待します。

{{< caution >}}
**重要:** Kubernetesホストがアクセスするために、ターゲットWWNへLUN (ボリューム) を割り当ててマスクするよう、前もってFC SAN Zoningを構成しなければなりません。
{{< /caution >}}

詳細は[FC example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/fibre_channel)を参照してください。

### flocker {#flocker}

[Flocker](https://github.com/ClusterHQ/flocker)はオープンソースのクラスタ化されたコンテナデータボリュームマネージャです。様々なストレージバックエンドを使ったデータボリュームの管理とオーケストレーションを提供します。

`flocker` VolumeはFolckerデータセットをPodにマウントできます。データセットがFlocker内に存在しなければ、Flocker CLIまたはFlocker APIを使って最初に作っておく必要があります。データセットが既に存在していれば、FlockerによってPodがスケジュールされるノードに再アタッチされます。このため、必要に応じてPod間でデータを手渡すことができます。

{{< caution >}}
**重要:** 使う前に、自身のFlockerをインストールしておかなければなりません。
{{< /caution >}}

詳細は[Flocker example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/flocker)を参照してください。

### gcePersistentDisk {#gcepersistentdisk}

`gcePersistentDisk` VolumeはGoogle Compute Engine (GC) の[Persistent Disk](http://cloud.google.com/compute/docs/disks)をPodにマウントします。Podが削除されると消える`emptyDir`とは異なり、PDの内容は保持され、Volumeはただアンマウントされるだけです。このため、PDはデータを事前投入でき、そのデータをPod間で手渡すことができます。

{{< caution >}}
**重要:** 使う前に、`gcloud`かGCE APIかUIを使ってPDを作っておかなければなりません。
{{< /caution >}}

`gcePersistentDisk`にはいくつかの制限があります。

* PodのノードはGCE VMで起動しなければならない
* これらのVMは、PDと同じプロジェクトとゾーンである必要あり

PDの機能は、複数の利用者が同時に読み込みのみでマウントできることです。このため、データセットを事前投入でき、必要なだけのPodに並列で提供できます。残念なことに、PDは読み書きモードでは単一の利用者にのみマウントできず、同時に書き込むことはできません。

ReplicationControllerで制御されるPodでのPDの使用は、PDが読み込みのみか、レプリカ数を0または1にしないと失敗します。

#### PDの作成 {#creationg-a-pd}

PodでGCE PDを使う前に、作成する必要があります。

```shell
gcloud compute disks create --size=500GB --zone=us-central1-a my-data-disk
```

#### 例 {#example-pod}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    # This GCE PD must already exist.
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4
```

#### Regional Persistent Disks
{{< feature-state for_k8s_version="v1.10" state="beta" >}}

[Regional Persistent Disks](https://cloud.google.com/compute/docs/disks/#repds)機能は同じリージョンの2つのゾーンで利用可能なPersistent Diskの作成を可能にします。この機能を使うためには、このVolumeがPersistentVolumeとしてプロビジョニングされなければなりません。Podから直接このVolumeを参照することはサポートされていません。

#### Regional PD PersistentVolumeの手動プロビジョニング {#manually-provisioning-a-regional-pd-persistentvolume}

[GCE PDのStorageClass](/docs/concepts/storage/storage-classes/#gce)を使った動的プロビジョニングが可能です。PersistentVolumeを作成する前に、PDを作成しなければなりません。

```shell
gcloud beta compute disks create --size=500GB my-data-disk
    --region us-central1
    --replica-zones us-central1-a,us-central1-b
```

PersistentVolumeスペックの例です。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-volume
  labels:
    failure-domain.beta.kubernetes.io/zone: us-central1-a__us-central1-b
spec:
  capacity:
    storage: 400Gi
  accessModes:
  - ReadWriteOnce
  gcePersistentDisk:
    pdName: my-data-disk
    fsType: ext4
```

### gitRepo (非推奨) {#gitrepo}

{{< warning >}}
**警告:** gitRepo Volumeタイプは非推奨です。git repoでコンテナをプロビジョニングするためには、gitを使ってリポジトリをcloneするInitContainerに[EmptyDir](#emptydir)をマウントして、その[EmptyDir](#emptydir)をPodのコンテナにマウントしてください。
{{< /warning >}}

`gitRepo` VolumeはVolumeプラグインとしてできることの一例です。これは、空のディレクトリをマウントし、Podにgitリポジトリをcloneします。将来、このようなVolumeは、Kubernetes APIを拡張するのではなく、さらに分離したモデルに移動する可能性があります。

gitRepo Volumeの例です。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: server
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /mypath
      name: git-volume
  volumes:
  - name: git-volume
    gitRepo:
      repository: "git@somewhere:me/my-git-repository.git"
      revision: "22f1d8406d464b0c0874075539c1f2e96c253775"
```

### glusterfs {#glusterfs}

`glusterfs` Volumeは[Glusterfs](http://www.gluster.org) (オープンソースのネットワークファイルシステム) のボリュームをPodにマウントできるようにします。Podが削除されると消える`emptyDir`とは異なり、`glusterfs` Volumeの内容は保持され、Volumeはただアンマウントされるだけです。このため、glusterfs Volumeはデータの事前投入ができ、Pod間でそのデータの手渡しができます。GlusterFSは同時に書き込むようマウントすることができます。

{{< caution >}}
**重要:** 使用する前にGlusterFSをインストールしておかなければなりません。
{{< /caution >}}

詳細は[GlusterFS example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/glusterfs)を参照してください。

### hostPath {#hostpath}

`hostPath` VolumeはホストノードのファイルシステムのファイルやディレクトリをPodにマウントします。これはほとんどのPodが必要とするものではありませんが、ある種のアプリケーションに対する強力な非常口として提供されています。

例えば、次のような場合に`hostPath`を使います。

* Dockerの内部にアクセスする必要のあるコンテナを実行する。`/var/lib/docker`の`hostPath`を使う
* コンテナでcAdvisorを実行する。`/sys`の`hostPath`を使う
* Podに与えられた`hostPath`が事前に存在すべきかどうか、作成すべきかどうか、何は存在すべきか、指定できるようにする

必要な`path`プロパティに加えて、ユーザは任意に`hostPath` Volumeのタイプを指定できます。

`type`フィールドでサポートされる値は次の通りです。

| 値 | ふるまい |
|:------|:---------|
| | 空文字列 (デフォルト) は後方互換性のためにあり、hostPath Volumeをマウントする前のチェックを実行しません |
| `DirectoryOrCreate` | 与えられたパスに何も存在しなければ、kubeletと同じグループとオーナーで、0755のパーミッションを持つ空のディレクトリを作成します |
| `Directory` | 与えられたパスにディレクトリが存在しなければなりません |
| `FileOrCreate` | 与えられたパスに何も存在しなければ、kubeletと同じグループとオーナーで、0644のパーミッションを持つ空のファイルを作成します |
| `File` | 与えられたパスにファイルが存在しなければなりません |
| `Socket` | 与えられたパスにUNIXソケットが存在しなければなりません |
| `CharDevice` | 与えられたパスにキャラクターデバイスが存在しなければなりません |
| `BlockDevice` | 与えられたパスにブロックデバイスが存在しなければなりません |

このVolumeのタイプは注意して使ってください。

* (podTemplateから作られたような) 同じ構成を持つPodが、ノードに異なるファイルがあるために、ノードによって異なるふるまいをする可能性がある
* Kubernetesがリソースを意識したスケジューリングを追加する場合、`hostPath`で使われるリソースはリソースとみなされない
* ホストで作成されたファイルとディレクトリはrootのみが書き込み可能である。[特権コンテナ](/docs/user-guide/security-context)でプロセスをrootとして実行するか、ファイルパーミッションを変更する必要があります。

#### 例 {#example-pod}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```

### iscsi {#iscsi}

`iscsi` Volumeは既存のiSCSI (SCSI over IP) ボリュームをPodにマウントできるようにします。Podが削除されると消える`emptyDir`とは異なり、`iscsi` Volumeの内容は保持され、Volumeはただアンマウントされるだけです。これにより、iscsi Volumeにデータを事前投入することができ、Pod間でそのデータを手渡すことができます。

{{< caution >}}
**重要:** 使用する前に、ボリューム作成済みのiSCSIサーバが必要です。
{{< /caution >}}

iSCSIの機能は同時に複数の利用者が読み込みのみでマウントできることです。これにより、データセットを事前投入しておき、必要なだけのPodへそのデータを並列に提供することができます。残念なことに、iSCSI Volumeは単一の利用者しか読み書きモードでマウントできず、同時に書き込むことはできません。

詳細は[iSCSI example](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/iscsi)を参照してください。

### local {#local}

{{< feature-state for_k8s_version="v1.10" state="beta" >}}

{{< note >}}
**メモ:** アルファ機能のPersistentVolume NodeAffinityアノテーションは非推奨となり、将来のリリースで削除されます。このアノテーションを使う既存のPersistentVolumeは、新しい`NodeAffinity`フィールドを使うよう、ユーザが更新しなければなりません。
{{< /note >}}

`local` Volumeはディスクやパーティション、ディレクトリといった、マウントされたローカルストレージデバイスを表現します。

Local Volumeは静的に作成されたPersistentVolumeとしてのみ利用できます。動的プロビジョニングはまだサポートされていません。

`hostPath` Volumeと比較して、Local VolumeはPodのノードへの手動スケジューリングなしに、永続的でポータブルな方法で使えます。システムは、PersistentVolumeのノードアフィニティを見ることで、Volumeのノード制約を把握します。

しかしながら、Local Volumeはノードの可用性に依存するので、すべてのアプリケーションに対して適してはいません。ノードが異常になれば、そのLocal Volumeにもアクセス不能になり、それを使うPodは実行できなくなります。Local Volumeを使うアプリケーションは、使用するディスクの耐久性に応じて、潜在的なデータロスと同様、この可用性の低下に対応しなければなりません。

以下は、`local` Volumeと`nodeAffinity`を使うPersistentVolumeスペックの例です。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  # volumeMode field requires BlockVolume Alpha feature gate to be enabled.
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

PersistentVolumeの`nodeAffinity`はLocal Volumeを使う時に必要です。これは、Kubernetesスケジューラが正しいノードでLocal Volumeを使うPodを正しくスケジュールできるようにします。

PersistentVolumeの`volumeMode`は、Local VolumeをRawブロックデバイスとして公開するために (デフォルト値である"Filesystem"のかわりに) "Block"を設定できます。`volumeMode`フィールドは`BlockVolume`アルファFeature Gateを有効にする必要があります。

Local Volumeを使う場合、`volumeBindingMode`を`WaitForFirstConsumer`に設定したStorageClassを作ることをおすすめします。[例](/docs/concepts/storage/storage-classes/#local)を見てください。Volume結合を遅らせることで、PersistentVolumeBinding結合の決定でも、ノードのリソース要求やノードセレクタ、Podアフィニティ、Pod逆アフィニティといった、Podが持つ可能性のある他のノード制約を確実に評価するようにします。

Local Volumeライフサイクルの管理を良くするために、外部の静的プロビジョナは別で実行することができます。このプロビジョナはまだ動的プロビジョニングをサポートしないことに注意してください。外部のローカルプロビジョナを実行する例は、[local volume provisioner user guide](https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume)を参照してください。

{{< note >}}
**メモ:** ボリュームライフサイクルを管理するために外部の静的プロビジョナを使わないのであれば、Local PersistentVolumeはユーザによる手動でのクリーンアップと削除が必要です。
{{< /note >}}

### nfs {#nfs}

`nfs` Volumeは既存のNFS (Network File System) 共有をPodにマウントできるようにします。Podが削除されると消える`emptyDir`とは異なり、`nfs` Volumeの内容は保持され、Volumeはただアンマウントされるだけです。このため、NFS Volumeにはデータを事前投入することができ、そのデータをPod間で手渡すことができます。NFSは同時に複数の書き込みをするようマウントできます。

{{< caution >}}
**重要:** 使う前に共有をエクスポートするよう実行しているNFSサーバが必要です。
{{< /caution >}}

詳細は[NFSの例](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/nfs)を参照してください。

### persistentVolumeClaim {#persistentvolumeclaim}

`persistentVolumeClaim` Volumeは[PersistentVolume](/ja/docs/concepts/storage/persistent-volumes/)をPodにマウントするために使われます。PersistentVolumeは、ユーザが特定のクラウド環境の詳細を知ることなく永続的なストレージ (GCE PersistentDiskやiSCSIなど) を"請求"する方法です。

詳細は[PersistentVolumesの例](/docs/concepts/storage/persistent-volumes/)を参照してください。

### projected {#projected}

`projected` Volumeはいくつかの既存のVolumeソースを同じディレクトリにマッピングします。

現在、以下のVolumeソースが利用できます。

- [`secret`](#secret)
- [`downwardAPI`](#downwardapi)
- [`configMap`](#configmap)
- `serviceAccountToken`

すべてのソースがPodと同じ名前空間にある必要があります。詳細は[all-in-one volume design document](https://github.com/kubernetes/community/blob/{{< param "githubbranch" >}}/contributors/design-proposals/node/all-in-one-volume.md)を参照してください。

Service Account Tokenをマッピングする機能はKubernetes 1.11で導入され、1.12でベータに昇格しました。この機能を1.11で有効にするためには、明示的に`TokenRequestProjection` [Feature Gate](/docs/reference/command-line-tools-reference/feature-gates/)をTrueに設定する必要があります。

#### Secretとdownward API、configmapを持つPodの例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
            - key: config
              path: my-group/my-config
```

#### デフォルトとは異なるパーミッションを持つ複数Secretを使うPodの例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - secret:
          name: mysecret2
          items:
            - key: password
              path: my-group/my-password
              mode: 511
```

各projected Volumeソースはスペックの`sources`以下に列挙されます。パラメータは2つの例外を除いてほぼ同じです。

* Secretに対して、`secretName`フィールドはconfigMapと一貫性を持たせるため、`name`に変更される
* `defaultMode`はマッピングされたレベルでのみ指定でき、各Volumeソースでは指定できない。しかしながら、上で示したように、各マッピングに対して`mode`を明示的に設定することができる

`TokenRequestProjection`機能が有効になっている場合、現在の[Service Account](/docs/reference/access-authn-authz/authentication/#service-account-tokens)のトークンをPodの指定したパスに注入することができます。以下が例です。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: token-vol
      mountPath: "/service-account"
      readOnly: true
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```

例のPodは注入されたServiceAccountトークンを含むprojected Volumeを持ちます。このトークンはPodコンテナがKubernetes APIにアクセスする場合などに使うことができます。`audience`フィールドにはトークンの利用対象を含みます。トークンの受信者はトークンを利用対象で指定されたIDで自身を証明しなければならず、そうしなければトークンは拒絶されます。このフィールドはオプションで、デフォルトはAPIサーバのIPです。

`expirationSeconds`はServiceAccountトークンの有効期限です。デフォルトは1時間で、少なくとも10分 (600秒) 以上を指定しなければなりません。管理者はAPIサーバの`--service-account-max-token-expiration`を指定することで最大値も制限することができます。`path`フィールドはprojected Volumeのマウントポイントからの相対パスを指定します。

{{< note >}}
**メモ:** projected Volumeソースを[subPath](#using-subpath) Volumeとして使うコンテナはこれらのVolumeソースへの変更を受信しません。
{{< /note >}}

### quobyte {#quobyte}

`quobyte` Volumeは既存の[Quobyte](http://www.quobyte.com)ボリュームをPodにマウントできるようにします。

{{< caution >}}
**重要:** これを使う前に、ボリュームを作成して実行しているQuobyteが必要です。
{{< /caution >}}

詳細は[Quobyteの例](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/quobyte)を参照してください。

### rbd {#rbd}

`rbd` Volumeは[Rados Block Device](http://ceph.com/docs/master/rbd/rbd/)ボリュームをPodにマウントできるようにします。Podが削除されると消える`emptyDir`と異なり、`rbd` Volumeの内容は保持され、Volumeはただアンマウントされるだけれす。そのため、RBD Volumeはデータを事前投入しておくことができ、Pod間で手渡すことができます。

{{< caution >}}
**重要:** RBDを使う前にCephをインストールして実行しておかなければなりません。
{{< /caution >}}

RBDの機能は、同時に複数の利用者から読み込みのみでマウントできます。そのため、データセットを事前投入しておき、必要なだけのPodに並列に提供できます。残念ながら、RBD Volumeは単体の利用者からしか読み書きモードでマウントできず、同時書き込みは許されません。

詳細は[RBDの例](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/rbd)を参照してください。

### scaleIO {#scaleio}

ScaleIOは既存のハードウェアに、スケーラブルなブロックネットワークストレージのクラスタを作るために使われます。`scaleIO` VolumeプラグインはデプロイされたPodに既存のScaleIO Volumeへアクセスできるようにします (もしくは、PersistentVolumeClaimに対して動的プロビジョニングできるようにします。詳細は[ScaleIO Persistent Volumes](/docs/concepts/storage/persistent-volumes/#scaleio)を参照してください)。

{{< caution >}}
**重要:** これを使う前にセットアップとボリューム作成の済んだScaleIOクラスタが必要です。
{{< /caution >}}

以下がScaleIOを使ったPod構成の例です。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-0
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: pod-0
    volumeMounts:
    - mountPath: /test-pd
      name: vol-0
  volumes:
  - name: vol-0
    scaleIO:
      gateway: https://localhost:443/api
      system: scaleio
      protectionDomain: sd0
      storagePool: sp1
      volumeName: vol-0
      secretRef:
        name: sio-secret
      fsType: xfs
```

詳細は[ScaleIOの例](https://github.com/kubernetes/examples/tree/{{< param "githubbranch" >}}/staging/volumes/scaleio)を参照してください。

### secret {#secret}

`secret` Volumeは、パスワードなどの機密情報をPodに渡すために使われます。SecretをKubernetes APIに格納し、Kubernetesと直接結合することなく、ファイルとしてそれらをマウントできます。`secret` Volumeはtmpfs (RAMベースのファイルシステム) がベースとなっているので、非揮発性のストレージには決して書き込まれません。

{{< caution >}}
**重要:** これを使う前に、Kubernetes APIでSecretを作成しなければなりません。
{{< /caution >}}

{{< note >}}
**メモ:** [subPath](#using-subpath) VolumeとしてSecretを使うContainerは、Secretの更新を受け取りません。
{{< /note >}}

Secretの詳細は[ここ](/docs/user-guide/secrets)で述べられています。

### storageOS {#storageos}

`storageos` Volumeは既存の[StorageOS](https://www.storageos.com)ボリュームをPodにマウントできるようにします。

StorageOSはKubernetes環境のコンテナとして実行し、クラスタ内のノードからローカルやアタッチ済みストレージにアクセスできるようにします。データはノード障害から守るために複製することができます。シンプロビジョニングと圧縮は使い勝手の改善とコストを削減を可能にします。

StorageOSはブロックストレージをコンテナに提供し、ファイルシステムを通じてアクセス可能になります。

StorageOSコンテナは64-bit Linuxが必要で、それ以外の依存関係はありません。フリーの開発者ライセンスが利用可能です。

{{< caution >}}
**重要:** ストレージ容量をのプールに使うノードや、StoregaOSボリュームにアクセスしたい各ノードでStorageOSコンテナを実行しなければなりません。インストールについては、[StorageOSドキュメント](https://docs.storageos.com)を参照してください。
{{< /caution >}}

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: redis
    role: master
  name: test-storageos-redis
spec:
  containers:
    - name: master
      image: kubernetes/redis:v1
      env:
        - name: MASTER
          value: "true"
      ports:
        - containerPort: 6379
      volumeMounts:
        - mountPath: /redis-master-data
          name: redis-data
  volumes:
    - name: redis-data
      storageos:
        # この`redis-vol01`ボリュームは`default`名前空間のStorageOSに既に存在しなければなりません。
        volumeName: redis-vol01
        fsType: ext4
```

動的プロビジョニングとPersistentVolumeClaimを含む詳細な情報は、[StorageOSの例](https://github.com/kubernetes/examples/blob/master/staging/volumes/storageos)を参照してください。

### vsphereVolume {#vspherevolume}

{{< note >}}
**必須条件:** vSphereクラウドプロバイダにKubernetesを構成しなければなりません。クラウドプロバイダ構成については、[vSphere getting started guide](https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/)を参照してください。
{{< /note >}}

`vsphereVolume`はvSphere VMDKボリュームをPodにマウントするために使います。Volumeの内容はアンマウントされても保持されます。VMFSとVSANデータストアの両方をサポートします。

{{< caution >}}
**重要:** Podで使う前に、以下のいずれかの方法でVMDKを作成しなければなりません。
{{< /caution >}}

#### VMDKボリュームの作成

次のいずれかの方法でVMDKを作成してください。

{{< tabs name="tabs_volumes" >}}
{{% tab name="vmkfstoolsと使う" %}}
最初にESXへsshで入り、以下のコマンドを使ってVMDKを作成します。

```shell
vmkfstools -c 2G /vmfs/volumes/DatastoreName/volumes/myDisk.vmdk
```
{{% /tab %}}
{{% tab name="vmware-vdiskmanagerを使う" %}}
以下のコマンドを使ってVMDKを作成します。

```shell
vmware-vdiskmanager -c -t 0 -s 40GB -a lsilogic myDisk.vmdk
```
{{% /tab %}}

{{< /tabs >}}

#### vSphere VMDKを使った構成の例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-vmdk
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-vmdk
      name: test-volume
  volumes:
  - name: test-volume
    # このVMDKボリュームは既に存在しなければなりません。
    vsphereVolume:
      volumePath: "[DatastoreName] volumes/myDisk"
      fsType: ext4
```

その他の例が[ここ](https://github.com/kubernetes/examples/tree/master/staging/volumes/vsphere)にあります。


## subPath の利用 {#using-subpath}

単一の Pod で1つの Volume を複数の用途で共有するのが便利なことがあります。`volumeMounts.subPath` プロパティは参照された Volume のルートではなくサブパスを指定するために使われます。

以下は単一の共有 Volume を使う LAMP スタック (Linux Apache Mysql PHP) の Pod の例です。HTML コンテンツは `html` フォルダにマッピングされ、データベースは `mysql` フォルダに格納されます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd" 
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

### 拡張された環境変数で subPath を使う {#using-subpath-with-expanded-environment-variables}

{{< feature-state for_k8s_version="v1.11" state="alpha" >}}

`subPath` ディレクトリ名は Downward API 環境変数から構成することもできます。この機能を使う前に `VolumeSubpathEnvExpansion` feature gate を有効にしなければなりません。

この例では、Pod は Downward API から Pod 名を使って、hostPath ボリューム `/var/log/pods`内のディレクトリ `pod1` を作るために `subPath` を使います。ホストディレクトリ `/var/log/pods/pod1` はコンテナの `/logs` にマウントされます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      subPath: $(POD_NAME)
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath: 
      path: /var/log/pods
```

## リソース {#resources}

`emptyDir` ボリュームのストレージメディア (ディスクやSSDなど) はkubelet root dir (通常は `/var/lib/kubelet`) を保持するファイルシステムのメディアによって決定されます。`emptyDir` や `hostPath` ボリュームが使えるスペースは制限されず、コンテナ間や Pod 間で隔離されません。

将来、[resource](/ja/docs/user-guide/compute-resources) スペックを使って、`emptyDir` と `hostPath` ボリュームがスペースの要求や、クラスタがいくつかのメディアを持っている場合に利用するメディアのタイプの選択ができるようになることを期待している。

## Out-of-Tree Volumeプラグイン {#out-of-tree-volume-plugins}

Out-of-Tree VolumeプラグインにはContainer Storage Interface (CSI)とFlexvolumeがあります。これらはストレージベンダがカスタムストレージプラグインをKubernetesのリポジトリに追加することなく作成できるようにします。

CSIとFlexvolumeが導入される以前、すべてのボリュームプラグイン (上で列挙したようなVolumeタイプ) は、コアのKubernetesバイナリにビルドされ、リンクされ、コンパイルされ、提供されており、コアKubernetes APIを拡張する"In-Tree"なものでした。このため、新しいストレージシステムをKubernetesに追加するために、コアKubernetesコードリポジトリに入るコードのチェックが必要でした。

CSIとFlexvolumeはVolumeプラグインをKubernetesのコードベースから独立して開発し、拡張機能としてKubernetesクラスタにデプロイ (インストール) できるようにします。

ストレージベンダ向けのOut-of-Tree Volumeプラグインの作成方法については、[このFAQ](https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md)を参照してください。

### CSI

{{< feature-state for_k8s_version="v1.10" state="beta" >}}

[Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md) (CSI) は、(Kubernetesのような) コンテナオーケストレーションシステムに対して、コンテナワークロードへ任意のストレージシステムを公開するための標準インタフェースを定義します。

詳細は[CSI design proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md)を読んでください。

CSIのサポートはKubernetes v1.9でアルファとして導入され、Kubernetes v1.10でベータになりました。

CSI互換ボリュームドライバがKubernetesクラスタにデプロイされると、ユーザはCSIドライバによって公開されているボリュームをアタッチやマウントなどするために`csi`ボリュームを使えます。

`csi`ボリュームタイプはPodからの直接参照をサポートせず、`PersistentVolumeClaim`オブジェクト経由でのみ参照できます。

ストレージ管理者はCSI永続ボリュームを構成するために以下のフィールドが利用可能です。

- `driver`: 使用するボリュームドライバの名前を指定する文字列の値です。この値は[CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#getplugininfo)で定義されている、CSIドライバによって返される`GetPluginInfoResponse`の値に対応しなければなりません。
  これは、KubernetesがどのCSIドライバを呼び出したのかを識別し、CSIドライバコンポーネントがCSIドライバに属するPVオブジェクトがどれなのかを識別するために使われます。
- `volumeHandle`: ボリュームを一意に識別する文字列の値です。この値は[CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume)で定義されている、CSIドライバによって返される`CreateVolumeResponse`の`volume.id`フィールドの値に対応しなければなりません。
  この値は、ボリュームを参照する際に、CSIボリュームドライバのすべての呼び出しの`volume_id`として渡されます。
- `readOnly`: ボリュームが読み出しのみとして "ControllerPublished" (アタッチ) されたかどうかを示す、任意の真偽値です。この値は`ControllerPublishVolumeRequest`の`readonly`フィールドとしてCSIドライバに渡されます。
- `fsType`: PVの`VolumeMode`が`FileSystem`であれば、このフィールドはマウントするために使うべきファイルシステムを指定するために使われます。ボリュームがフォーマットされておらず、フォーマットがサポートされていれば、この値はボリュームをフォーマットするために使われます。
  この値は`ControllerPublishVolumeRequest`, `NodeStageVolumeRequest`, `NodePublishVolumeRequest`の`VolumeCapability`フィールド経由でCSIドライバに渡されます。
- `volumeAttributes`: ボリュームの静的なプロパティを指定する文字列から文字列へのマッピングです。このマッピングは[CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume)で定義されている、CSIドライバによって返される`CreateVolumeResponse`の`volume.attributes`フィールドのマッピングと対応しなければなりません。
  このマッピングは`ControllerPublishVolumeRequest`, `NodeStageVolumeRequest`, `NodePublishVolumeRequest`の`volume_attributes`フィールド経由でCSIドライバに渡されます。
- `controllerPublishSecretRef`: CSIの`ControllerPublishVolume`と`controllerUnpublishVolume`呼び出しを完了させるためにCSIドライバに渡される、機密情報を含むSecretオブジェクトへの参照です。このフィールドは任意で、Secretが必要なければ空でかまいません。Secretオブジェクトが複数のSecretを含むのであれば、すべてのSecretが渡されます。
- `nodeStageSecretRef`: CSIの`NodeStageVolume`呼び出しを完了させるためにCSIドライバに渡される、機密情報を含むSecretオブジェクトへの参照です。このフィールドは任意で、Secretが必要なければ空でかまいません。Secretオブジェクトが複数のSecretを含むのであれば、すべてのSecretが渡されます。
- `nodePublishSecretRef`: CSIの`NodePublishVolume`呼び出しを完了させるためにCSIドライバに渡される、機密情報を含むSecretオブジェクトへの参照です。このフィールドは任意で、Secretが必要なければ空でかまいません。Secretオブジェクトが複数のSecretを含むのであれば、すべてのSecretが渡されます。

#### CSIのRawブロックボリュームサポート

{{< feature-state for_k8s_version="v1.11" state="alpha" >}}

Version 1.11から、CSIはRawブロックボリュームのサポートを導入しました。これはKubernetesの以前のバージョンで導入されたRawブロックボリューム機能に依存します。この機能は、ベンダに対してKubernetesaワークロードのRawブロックボリュームをサポートする外部のCSIドライバの実装を可能にします。

CSIブロックボリュームのサポートはFeature Gateで管理されていて、デフォルトでは無効になっています。ブロックボリュームサポートを有効にしてCSIを実行するには、クラスタ管理者が以下のFeature Gateフラグを使って、各Kubernetesコンポーネントに対してこの機能を有効にしなければなりません。

```
--feature-gates=BlockVolume=true,CSIBlockVolume=true
```

[setup your PV/PVC with raw block volume support](/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)を参照してください。

### Flexvolume

Flexvolumeは、Kubernetes Version 1.2 (CSIよりも前) から存在するOut-of-Treeプラグインインタフェースです。ドライバのインタフェースにはexecベースモデルを使います。Flexvolumeドライバのバイナリは各ノードの定義済みボリュームプラグインパスにインストールされなければなりません。

PodはFlexvolumeドライバと`flexvolume` In-Treeプラグインを通じてやりとりします。詳細は[here](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md)を参照してください。

## マウント伝播 {#mount-propagation}

マウント伝播はコンテナによってマウントされたボリュームを、同じ Pod の他のコンテナもしくは同じノードの他の Pod にも、共有することを可能にします。

ボリュームのマウント伝播は Container.volumeMounts の `mountPropagation` フィールドで制御されます。以下の値をとります。

 * `None` - このボリュームマウントはこのボリュームにマウントされる後続のマウントもしくはホストのサブディレクトリを受け取りません。同様に、コンテナによって作成されたマウントはホストには見えません。これがデフォルトのモードです。

   このモードは [Linux kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) に記載されている `private` マウント伝播と等価です。

 * `HostToContainer` - このボリュームマウントはこのボリュームにマントされるすべての後続のマウントもしくはサブディレクトリを受け取ります。
 
   言い換えると、ホストはこのボリュームマウントの中身をすべてマウントするのであれば、コンテナはそこにマウントされているように見えます。
   
   同様に、同じボリュームに `Bidirectional` マウント伝播を持つ Pod がそこのすべてをマウントするのであれば、 `HostToContainer` マウント伝播を持つコンテナはそれが見えます。
   
   このモードは [Linux kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) に記載されている `rslave` マウント伝播と等価です。
   
 * `Bidirectonal` - このボリュームマウントは `HostToContainer` マウントと同様にふるまいます。加えて、コンテナによって作成されたすべてのボリュームマウントはホストと同じボリュームを使うすべての Pod のコンテナに逆伝播します。
 
   このモードの典型的なユースケースは `FlexVolume` もしくは `CSI` ドライバをもつ Pod もしくは、`hostPath` ボームを使うホストの何かをマウントする必要のある Pod です。
   
   このモードは [Linux kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) に記載されている `rshared` マウント伝播と等価です。

{{< caution >}}
**注意** `Bidirectional` マウント伝播は危険である可能性があります。ホストのオペレーティングシステムにダメージを与えることができるので、権限のあるコンテナにのみ許可されます。Linux カーネルの動作に精通していることを強く勧めます。加えて、Pod 内のコンテナによって作成されたボリュームマウントは削除されるコンテナによって削除 (アンマウント) されなければなりません。
{{< /caution >}}

### 構成 {#configuration}

いくつかの Deployment (CoreOS, RedHat/CentOS, Ububtu)) でマウント伝播が正しく動作する前に、以下のように Docker でマウント共有を正しく構成しておかなければなりません。

Docker の `systemd` サービスファイルを編集してください。`MountFlags` を以下のように設定します。

```shell
MountFlags=shared
```

もしくは、`MountFlags=slave` があれば削除します。そして、Docker デーモンを再起動します。

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

{{% /capture %}}

{{% capture whatsnext %}}
* [Persistent Volume を持つ WordPress と MySQL のデプロイ](/ja/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/) の例に進みましょう
{{% /capture %}}
