---
title: "例: PersistentVolumeを持つWordPressとMySQLのデプロイ"
content_template: templates/tutorial
weight: 20
---

{{% capture overview %}}
このチュートリアルではMinikubeを使ったWordPressサイトとMySQLデータベースのデプロイ方法を示します。両方のアプリケーションがデータを格納するためにPersistentVolumeとPersistentVolumeClaimを使います。

[PersistentVolume](/ja/docs/concepts/storage/persistent-volumes/) (PV)は、管理者が手動でプロビジョンするか、[StorageClass](/docs/concepts/storage/storage-classes)を使ってKubernetesが動的にプロビジョンした、クラスタのストレージの一部です。[PersistentVolumeClaim](/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) (PVC)はユーザによるストレージのリクエストです。PersistentVolumeとPersistentVolumeClaimはPodのライフサイクルとは独立しており、Podが再起動や再スケジューリング、削除されたとしても、データは保持されます。

{{< warning >}}
**警告:** このデプロイは本番のユースケースには適していません。本番でWordPressをデプロイするには[WordPress Helm Chart](https://github.com/kubernetes/charts/tree/master/stable/wordpress)を使うことを検討してください。
{{< /warning >}}

{{< note >}}
**メモ:** このチュートリアルで提供するファイルはGA Deployment APIを使い、Kubernetes 1.9以降が対象です。以前のバージョンでこのチュートリアルを使いたい場合、APIバージョンを適切に更新するか、このチュートリアルの以前のバージョンを参照してください。
{{< /note >}}

{{% /capture %}}

{{% capture objectives %}}
* PersistentVolumeClaimとPersistentVolumeを作る
* Secretを作る
* MySQLをデプロイする
* WordPressをデプロイする
* クリーンアップする
{{% /capture %}}

{{% capture prerequisites %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

以下の構成ファイルをダウンロードしてください。

1. [mysql-deployment.yaml](/ja/examples/application/wordpress/mysql-deployment.yaml)

2. [wordpress-deployment.yaml](/ja/examples/application/wordpress/wordpress-deployment.yaml)

{{% /capture %}}

{{% capture lessoncontent %}}

## PersistentVolumeClaimとPersistentVolumeを作成する {#create-persistentvolumeclaims-and-persistentvolumes}

MySQLとWordpressはそれぞれ、データの格納にPersistentVolumeが必要です。PersistentVolumeClaimはデプロイの段階で作成します。

多くのクラスタ環境ではデフォルトのStorageClassがインストールされています。PersistentVolumeClaimでStorageClassが指定されていない場合、クラスタのデフォルトStorageClassが使われます。

PersistentVolumeClaimが作成されると、StorageClassの構成に基づいて、PersistentVolumeが動的にプロビジョニングされます。

{{< warning >}}
**警告:** ローカルクラスタでは、デフォルトStorageClassに`hostPath`プロビジョナを使います。`hostPath`ボリュームは開発とテスト目的にのみ適しています。`hostPath`ボリュームでは、データはPodがスケジュールされたノードの`/tmp`に格納され、ノード間を移動したりはしません。もしPodが削除され、ほかのノードにスケジュールされるか、ノードが再起動されると、データは失われます。
{{< /warning >}}

{{< note >}}
**メモ:** `hostPath`プロビジョナを使う必要のあるクラスタを立ち上げているのなら、`controller-manager`コンポーネントで`--enable-hostpath-provisioner`フラグを設定しておかなければなりません。
{{< /note >}}

{{< note >}}
**メモ:** Google Kubernetes Engineで動いているKubernetesクラスタがあるのなら、[このガイド](https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk)に従ってください。
{{< /note >}}

## MySQLパスワードのSecretを作成する {#create-a-secret-for-mysql-password}

[Secret](/docs/concepts/configuration/secret/)はパスワードやキーのような機密情報の一部を格納するオブジェクトです。マニフェストファイルにはすでにSecretを使うように構成されていますが、自身のSecretを作成する必要があります。

1. 以下のコマンドでSecretオブジェクトを作成する。`YOUR_PASSWORD`は使いたいパスワードに置き換える

      ```shell
      kubectl create secret generic mysql-pass --from-literal=password=YOUR_PASSWORD
      ```

2. 以下のコマンドを実行し、Secretが存在するか確認する

      ```shell
      kubectl get secrets
      ```

      出力は次のようになる

      ```
      NAME                  TYPE                    DATA      AGE
      mysql-pass            Opaque                  1         42s
      ```

{{< note >}}
**メモ:** 漏洩からSecretを守るため、`get`や`describe`はその内容を表示しません。
{{< /note >}}

## MySQLをデプロイする {#deploy-mysql}

以下のマニフェストは単一インスタンスのMySQL Deploymentについて記述しています。MySQLコンテナはPersistentVolumeを/var/lib/mysqlにマウントします。`MYSQL_ROOT_PASSWORD`環境変数はSecretからデータベースのパスワードを設定します。

{{< codenew file="application/wordpress/mysql-deployment.yaml" >}}

1. `mysql-deployment.yaml`ファイルからMySQLをデプロイする

      ```shell
      kubectl create -f https://k8s.io/examples/application/wordpress/mysql-deployment.yaml
      ```
2. PersistentVolumeが動的にプロビジョニングされているか確認する。PVがプロビジョニングされ結合されるまでに数分かかることがあることに注意

      ```shell
      kubectl get pvc
      ```

      出力は次のようになる

      ```
      NAME             STATUS    VOLUME                                     CAPACITY ACCESS MODES   STORAGECLASS   AGE
      mysql-pv-claim   Bound     pvc-91e44fbf-d477-11e7-ac6a-42010a800002   20Gi     RWO            standard       29s
      ```

3. 以下のコマンドを実行し、Podが起動しているか確認する

      ```shell
      kubectl get pods
      ```

      {{< note >}}**メモ:** Podのステータスが`RUNNNING`になるまで数分かかることがある{{< /note >}}

      出力は次のようになる。

      ```
      NAME                               READY     STATUS    RESTARTS   AGE
      wordpress-mysql-1894417608-x5dzt   1/1       Running   0          40s
      ```

## WordPressをデプロイする {#deploy-wordpress}

以下のマニフェストは単一インスタンスのWordPress DeploymentとServiceについて記述しています。永続ストレージのためのPersistentVolumeやパスワードのためのSecretのような同じ機能を多く使っています。しかし、`type: LoadBalancer`の設定は異なっています。この設定はクラスタの外からのトラフィックにWordPressを公開するものです。

{{< codenew file="application/wordpress/wordpress-deployment.yaml" >}}

1. `wordpress-deployment.yaml`ファイルから WordPress ServiceとDeploymentを作成する

      ```shell
      kubectl create -f https://k8s.io/examples/application/wordpress/wordpress-deployment.yaml
      ```

2. PersistentVolumeが動的にプロビジョニングされていることを確認する

      ```shell
      kubectl get pvc
      ```

      {{< note >}}**メモ:** PVがプロビジョニングされ結合されるまで数分かかることがある {{< /note >}}
      
      出力は次のようになる

      ```
      NAME             STATUS    VOLUME                                     CAPACITY ACCESS MODES   STORAGECLASS   AGE
      wp-pv-claim      Bound     pvc-e69d834d-d477-11e7-ac6a-42010a800002   20Gi     RWO            standard       7s
      ```

3. 以下のコマンドを実行し、Serviceが稼動しているか確認する

      ```shell
      kubectl get services wordpress
      ```

      出力は次のようになる
      
      ```
      NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
      wordpress   ClusterIP   10.0.0.89    <pending>     80:32406/TCP   4m
      ```

      {{< note >}}**メモ:** Minikubeは`NodePort`を通じてのみServiceを公開できる。EXTERNAL-IPは常に保留になる{{< /note >}}

4. WordPress ServiceのIPアドレスを得るために、次のコマンドを実行する

      ```shell
      minikube service wordpress --url
      ```

      出力は次のようになる

      ```
      http://1.2.3.4:32406
      ```

5. IPアドレスをコピーし、ブラウザに貼り付けてサイトを閲覧する

   以下のスクリーンショットのような、WordPressのセットアップページが見られるはずである
   
   ![wordpress-init](https://raw.githubusercontent.com/kubernetes/examples/master/mysql-wordpress-pd/WordPress.png)

{{< warning >}}
**警告:** このWordPressインストールページを放置しないでください。ほかのユーザに見つかると、ウェブサイトをセットアップされ、悪意のあるコンテンツを配信するために使われる可能性があります。<br/><br/>ユーザ名とパスワードを作成してWordPressをインストールするか、インスタンスを削除してください。
{{< /warning >}}

{{% /capture %}}

{{% capture cleanup %}}

1. Secretを削除するために、以下のコマンドを実行する

      ```shell
      kubectl delete secret mysql-pass
      ```

2. すべてのDeploymentとServiceを削除するために、以下のコマンドを実行する

      ```shell
      kubectl delete deployment -l app=wordpress
      kubectl delete service -l app=wordpress
      ```

3. PersistentVolumeClaimを削除するために、次のコマンドを実行する。動的にプロビジョニングされたPersistentVolumeは自動的に削除される

      ```shell
      kubectl delete pvc -l app=wordpress
      ```

{{% /capture %}}

{{% capture whatsnext %}}

* [Introspection and Debugging](/docs/tasks/debug-application-cluster/debug-application-introspection/)についてさらに学ぶ
* [Job](/ja/docs/concepts/workloads/controllers/jobs-run-to-completion/)についてさらに学ぶ
* [Port Forwarding](/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)についてさらに学ぶ
* [コンテナへのShellを得る](/docs/tasks/debug-application-cluster/get-shell-running-container/)方法について学ぶ

{{% /capture %}}
