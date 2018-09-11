---
title: Volumes
content_template: templates/concept
weight: 10
---

{{% capture overview %}}

コンテナのディスク上のファイルは一時的なものであり、これがコンテナで実行される重要なアプリケーションにとって問題となることがあります。まず、コンテナがクラッシュすると、kubelet はそれを再起動させるが、ファイルは失われます --- コンテナはクリーンな状態で起動されます。次に、コンテナを `Pod` 内で一緒に起動する場合、これらのコンテナ間でたびたびファイルを共有する必要があります。Kubernetes の `Volume` 抽象化はこれらの問題を両方とも解決します。

[Pod](/ja/docs/user-guide/pods) について精通していることをお勧めします。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## 背景

Docker も [volumes](https://docs.docker.com/engine/admin/volumes/) の概念を持ちますが、いくぶんゆるく管理されます。Docker では、volume は単純にディスク上のディレクトリか他のコンテナです。生存時間は管理されず、つい最近までローカルディスクの volume があるだけでした。現在は Docker は volume ドライバーを提供していますが、今でもその機能は非常に制限されています (たとえば、Docker 1.7 ではコンテナに対して 1 つの volume ドライバーしか許されず、volume にパラメータを渡す方法がありません)。

他方で、Kubernetes の Volume は明示的な生存期間を持ちます --- Volume を含む Pod と同じです。その結果、Volume は Pod 内で実行するコンテナよりも長生きで、データはコンテナの再起動を経ても保持されます。もちろん、Podが存在しなくなれば、Volume も存在しなくなります。おそらくこれより重要なのは、Kubernetes が多くのタイプの Volume をサポートし、Pod はこれらを同時に使えるということです。

根本的には、Volume は単なるディレクトリで、何らかのデータが中にある可能性があり、Pod のコンテナがアクセス可能です。そのディレクトリがどのようにしてやって来るのか、支えとなるメディアやその内容は使用する Volume タイプによって決められます。

Volume を使うためには、Pod は使用する Volume を指定し (`.spec.volumes` フィールド)、それらのコンテナへのマウント先を指定します (`.spec.containers.volumeMounts` フィールド)。

コンテナのプロセスは [Docker イメージ](https://docs.docker.com/userguide/dockerimages/)と Volume で構成されたファイルシステムビューを見ます。Docker イメージはファイルシステム階層のルートで、volume はイメージ内の指定されたパスにマウントされます。Volume は他の Volume 上にマウントできず、他の Volume へのハードリンクも持てません。Pod 内の各コンテナは各 Volume のマウント先を独立して指定しなければなりません。

## Volume のタイプ

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

## subPath の利用

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

### 拡張された環境変数で subPath を使う

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

## リソース

`emptyDir` ボリュームのストレージメディア (ディスクやSSDなど) はkubelet root dir (通常は `/var/lib/kubelet`) を保持するファイルシステムのメディアによって決定されます。`emptyDir` や `hostPath` ボリュームが使えるスペースは制限されず、コンテナ間や Pod 間で隔離されません。

将来、[resource](/ja/docs/user-guide/compute-resources) スペックを使って、`emptyDir` と `hostPath` ボリュームがスペースの要求や、クラスタがいくつかのメディアを持っている場合に利用するメディアのタイプの選択ができるようになることを期待している。

## マウント伝播

{{< feature-state for_k8s_version="v1.10" state="beta" >}}

マウント伝播はコンテナによってマウントされたボリュームを、同じ Pod の他のコンテナもしくは同じノードの他の Pod にも、共有することを可能にします。

`MountPropagation` 機能が無効であるか、Pod が特定のマウント伝播を明示的に指定しなければ、Pod のコンテナにマウントされたボリュームは伝播しません。したがって、コンテナは [Linux kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)に書かれている `private` マウント伝播を用いて実行されます。

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

### 構成

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

{{% capture whatsnext %}}
* [Persistent Volume を持つ WordPress と MySQL のデプロイ](/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/) の例に進みましょう
{{% /capture %}}
