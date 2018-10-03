---
title: 単一インスタンスのステートフルアプリケーションの実行
content_template: templates/tutorial
weight: 20
---

{{% capture overview %}}

このページではPersistentVolumeとDeploymentを使ったKubernetesでの単一インスタンスのステートフルアプリケーションの実行方法を示します。アプリケーションはMySQLです。

{{% /capture %}}

{{% capture objectives %}}

* ディスクを参照するPersistentVolumeを作成する
* MySQL Deploymentを作成する
* クラスタの他のPodにDNS名を使ってMySQLを公開する

{{% /capture %}}

{{% capture prerequisites %}}

* {{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

* {{< include "default-storage-class-prereqs.md" >}}

{{% /capture %}}

{{% capture lessoncontent %}}

## MySQLのデプロイ {#deploy-mysql}

KubernetesのDeploymentを作成し、PersistentVolumeClaimを使って既存のPersistentVolumeに接続することで、ステートフルアプリケーションを実行できます。例えば、このYAMLファイルはMySQLを実行し、PersistentVolumeClaimを参照するDeploymentについて記述しています。このファイルは/var/lib/mysqlへのボリュームマウントを定義し、20Gのボリュームを探すPersistentVolumeClaimを作成します。このClaimは必要条件を満たす既存のボリュームが存在するか、動的プロビジョナーによって満たされます。

メモ: パスワードがconfig yamlで定義されていますが、これはセキュアではありません。セキュアな解決法は[Kubernetes Secret](/docs/concepts/configuration/secret/)を参照してください。

{{< codenew file="application/mysql/mysql-deployment.yaml" >}}
{{< codenew file="application/mysql/mysql-pv.yaml" >}}

1. YAMLファイルのPVとPVCをデプロイする

        kubectl create -f https://k8s.io/examples/application/mysql/mysql-pv.yaml

2. YAMLファイルの内容をデプロイする

        kubectl create -f https://k8s.io/examples/application/mysql/mysql-deployment.yaml

3. Deploymentについての情報を表示する

        kubectl describe deployment mysql

        Name:                 mysql
        Namespace:            default
        CreationTimestamp:    Tue, 01 Nov 2016 11:18:45 -0700
        Labels:               app=mysql
        Annotations:          deployment.kubernetes.io/revision=1
        Selector:             app=mysql
        Replicas:             1 desired | 1 updated | 1 total | 0 available | 1 unavailable
        StrategyType:         Recreate
        MinReadySeconds:      0
        Pod Template:
          Labels:       app=mysql
          Containers:
           mysql:
            Image:      mysql:5.6
            Port:       3306/TCP
            Environment:
              MYSQL_ROOT_PASSWORD:      password
            Mounts:
              /var/lib/mysql from mysql-persistent-storage (rw)
          Volumes:
           mysql-persistent-storage:
            Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
            ClaimName:  mysql-pv-claim
            ReadOnly:   false
        Conditions:
          Type          Status  Reason
          ----          ------  ------
          Available     False   MinimumReplicasUnavailable
          Progressing   True    ReplicaSetUpdated
        OldReplicaSets:       <none>
        NewReplicaSet:        mysql-63082529 (1/1 replicas created)
        Events:
          FirstSeen    LastSeen    Count    From                SubobjectPath    Type        Reason            Message
          ---------    --------    -----    ----                -------------    --------    ------            -------
          33s          33s         1        {deployment-controller }             Normal      ScalingReplicaSet Scaled up replica set mysql-63082529 to 1

4. Deploymentによって作成されたPodを列挙する

        kubectl get pods -l app=mysql

        NAME                   READY     STATUS    RESTARTS   AGE
        mysql-63082529-2z3ki   1/1       Running   0          3m

5. PersistentVolumeClaimを調べる

        kubectl describe pvc mysql-pv-claim

        Name:         mysql-pv-claim
        Namespace:    default
        StorageClass:
        Status:       Bound
        Volume:       mysql-pv-volume
        Labels:       <none>
        Annotations:    pv.kubernetes.io/bind-completed=yes
                        pv.kubernetes.io/bound-by-controller=yes
        Capacity:     20Gi
        Access Modes: RWO
        Events:       <none>

## MySQLインスタンスへのアクセス {#accessing-the-mysql-instance}

前述のYAMLファイルはクラスタの他のPodがデータベースにアクセスすることを許可するサービスを作成します。サービスのオプションである `clusterIP: None`はサービスのDNS名を直接PodのIPアドレスに解決させるものです。サービスの背後に1つのPodしかなく、Podの数を増やすつもりがないのであれば、最適な選択肢です。

MySQLクライアントをサーバに接続するために実行します。

```
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -ppassword
```

このコマンドは、MySQLコマンドを実行するPodを作成し、サービスを通じてサーバに接続します。接続されれば、ステートフルなMySQLサーバが起動していることがわかります。

```
Waiting for pod default/mysql-client-274442439-zyp6i to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.

mysql>
```

## 更新 {#updating}

イメージやDeploymentの他の部分の変更は、通常通り`kubectl apply`コマンドで更新できます。ステートフルアプリに特有の注意事項をいくつか挙げます。

* アプリをスケールしてはいけません。このセットアップは単一インスタンスアプリのみのものです。PersistentVolumeは1つのPodからしかマウントできません。クラスタ化したステートフルアプリについては[StatefulSetのドキュメント](/ja/docs/concepts/workloads/controllers/statefulset/)を参照してください。
* Deploymentの構成YAMLファイルでは`strategy:` `type: Recreate`を使ってください。これはKubernetesにローリングアップデートを _しない_ ように指示します。一度に複数のPodを起動できないので、ローリングアップデートは動作しません。`Recreate`戦略は最初のPodを停止させてから、構成が更新された新しいPodを作成します。

## Deploymentの削除 {#deleting-a-deployment}

名前を使ってデプロイされたオブジェクトを削除します。

```
kubectl delete deployment,svc mysql
kubectl delete pvc mysql-pv-claim
kubectl delete pv mysql-pv-volume
```

PersistentVolumeを手動でプロビジョニングしていれば、その中のリソースも同様に、手動で削除する必要があります。動的プロビジョナーを使っていれば、PersistentVolumeClaimが削除されると、PersistentVolumeも自動的に削除されます。動的プロビジョナーの中にはPersistentVolumeが削除されると、中のリソースも解放するものもあります。

{{% /capture %}}


{{% capture whatsnext %}}

* [Deploymentオブジェクト](/docs/concepts/workloads/controllers/deployment/)についてさらに学びます

* [アプリケーションのデプロイ](/docs/user-guide/deploying-applications/)についてさらに学びます

* [kubectl runのドキュメント](/docs/reference/generated/kubectl/kubectl-commands/#run)

* [Volume](/ja/docs/concepts/storage/volumes/)と[PersistentVolume](/ja/docs/concepts/storage/persistent-volumes/)

{{% /capture %}}

