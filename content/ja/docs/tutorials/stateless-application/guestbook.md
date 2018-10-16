---
title: "例: Redisを使ったPHP Guestbookアプリケーションのデプロイ"
content_template: templates/tutorial
weight: 20
---

{{% capture overview %}}
このチュートリアルは、Kubernetesと[Docker](https://www.docker.com/)を使った、シンプルなマルチティアWebアプリケーションをビルドしデプロイする方法を示します。この例は以下のコンポーネントで構成されます。

* Guestbookの内容を格納するシングルインスタンスの[Redis](https://redis.io/)マスタ
* 読み出しを提供するマルチ[レプリカRedis](https://redis.io/topics/replication)インスタンス
* マルチWebフロントエンドインスタンス

{{% /capture %}}

{{% capture objectives %}}
* Redisマスタを起動する
* Redisスレーブを起動する
* Guestbookフロントエンドを起動する
* フロントエンドServiceを公開、閲覧する
* クリーンアップする
{{% /capture %}}

{{% capture prerequisites %}}

{{< include "task-tutorial-prereqs.md" >}}

{{< version-check >}}

{{% /capture %}}

{{% capture lessoncontent %}}

## Redisマスタの起動 {#start-up-the-redis-master}

このGuestbookアプリケーションはデータを保存するためにRedisを使います。データはRedisマスタインスタンスに書き込み、複数のRedisスレーブインスタンスから読み出します。

### RedisマスタDeploymentの作成 {#creating-the-redis-master-deployment}

下で示すマニフェストファイルは単一レプリカのRedisマスタPodを実行するDeploymentコントローラを指定します。

{{< codenew file="application/guestbook/redis-master-deployment.yaml" >}}

1. ターミナルを起動し、マニフェストファイルをダウンロードする
2. `redis-master-deploymnet.yaml`ファイルのRedisマスタDeploymentを適用する

      ```shell
      kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-deployment.yaml
      ```
3. RedisマスタPodが起動しているかどうか確かめるために、Podのリストを問い合わせる

      ```shell
      kubectl get pods
      ```

       出力は次のようになる

      ```shell
      NAME                            READY     STATUS    RESTARTS   AGE
      redis-master-1068406935-3lswp   1/1       Running   0          28s
      ```
4. RedisマスタPodからのログを見るために次のコマンドを実行する

     ```shell
     kubectl logs -f POD-NAME
     ```
{{< note >}}
**メモ:** POD-NAMEはPodの名前で置き換えてください。
{{< /note >}}

### RedisマスタServiceの作成 {#creating-the-redis-master-service}

Guestbookアプリケーションはデータを書き込むためにRedisマスタと通信します。RedisマスタPodへのトラフィックをプロキシするために[Service](/docs/concepts/services-networking/service/)を適用する必要があります。ServiceはPodへのアクセスのポリシを定義します。

{{< codenew file="application/guestbook/redis-master-service.yaml" >}}

1. `redis-master-service.yaml`ファイルのRedisマスタServiceを適用する

      ```shell
      kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-service.yaml
      ```
2. RedisマスタServiceが起動しているか確かめるために、Serviceのリストを問い合わせる

      ```shell
      kubectl get service
      ```

       出力は次のようになる

      ```shell
      NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
      kubernetes     ClusterIP   10.0.0.1     <none>        443/TCP    1m
      redis-master   ClusterIP   10.0.0.151   <none>        6379/TCP   8s
      ```

{{< note >}}
**メモ:** このマニフェストファイルは、前に定義したラベルにマッチするラベルのセットを持つ`redis-master`という名前のServiceを作成します。したがって、ServiceはネットワークトラフィックをRedisマスタPodにルーティングします。
{{< /note >}}

## Redisスレーブの起動 {#start-up-the-redis-slaves}

Redisマスタは単一のPodでしたが、Redisスレーブを追加することでネットワークの要求に合わせて可用性を高めることができます。

### RedisスレーブDeploymentの作成 {#creating-the-redis-slave-deployment}

Deploymentのスケールのベースはマニフェストファイルで設定します。この場合、Deploymentオブジェクトは2つのレプリカを指定します。

まだレプリカが実行していなければ、このDeploymentは2つのレプリカを起動します。逆に、3つ以上のレプリカが実行していれば、2つになるまでスケールダウンします。

{{< codenew file="application/guestbook/redis-slave-deployment.yaml" >}}

1. `redis-slave-deployment.yaml`ファイルのRedisスレーブDeploymentを適用する

      ```shell
      kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-deployment.yaml
      ```

2. RedisスレーブPodが起動しているか確かめるために、Podのリストを問い合わせる

      ```shell
      kubectl get pods
      ```

       出力は次のようになる

      ```shell
      NAME                            READY     STATUS              RESTARTS   AGE
      redis-master-1068406935-3lswp   1/1       Running             0          1m
      redis-slave-2005841000-fpvqc    0/1       ContainerCreating   0          6s
      redis-slave-2005841000-phfv9    0/1       ContainerCreating   0          6s
      ```

### RedisスレーブServiceの作成 {#creating-the-redis-slave-service}

Guestbookアプリケーションはデータの読み出しにRedisスレーブと通信する必要があります。Redisスレーブを発見可能にするため、Serviceをセットアップする必要があります。ServiceはPodの集合に対する透過的なロードバランシングを提供します。

{{< codenew file="application/guestbook/redis-slave-service.yaml" >}}

1. `redis-slave-service.yaml`ファイルのRedisスレーブServiceを適用する

      ```shell
      kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-service.yaml
      ```

2. RedisスレーブServiceが起動しているか確かめるために、Serviceのリストを問い合わせる
      ```shell
      kubectl get services
      ```

       出力は次のようになる

      ```
      NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
      kubernetes     ClusterIP   10.0.0.1     <none>        443/TCP    2m
      redis-master   ClusterIP   10.0.0.151   <none>        6379/TCP   1m
      redis-slave    ClusterIP   10.0.0.223   <none>        6379/TCP   6s
      ```

## Guestbookフロントエンドのセットアップと公開 {#set-up-and-expose-the-guestbook-frontend}

このGuestbookアプリケーションはPHPで書かれたHTTPリクエストを提供するWebフロントエンドです。リクエストの書き込みに`redis-master` Serviceに接続し、リクエストの読み出しに`redis-slave` Serviceに接続するよう構成されています。

### GuestbookフロントエンドDeploymentの作成 {#creating-the-guestbook-frontend-deployment}

{{< codenew file="application/guestbook/frontend-deployment.yaml" >}}

1. `frontend-deployment.yaml`ファイルのフロントエンドDeploymentを適用する

      ```shell
      kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml
      ```
2. 3つのフロントエンドレプリカが起動しているか確認するために、Podのリストを問い合わせる

      ```shell
      kubectl get pods -l app=guestbook -l tier=frontend
      ```

       出力は次にようになる

      ```
      NAME                        READY     STATUS    RESTARTS   AGE
      frontend-3823415956-dsvc5   1/1       Running   0          54s
      frontend-3823415956-k22zn   1/1       Running   0          54s
      frontend-3823415956-w9gbt   1/1       Running   0          54s
      ```

### フロントエンドServiceの作成 {#creating-the-frontend-service}

適用した`redis-slave`と`redis-master`のServiceは、デフォルトのServiceタイプが[ClusterIP](/docs/concepts/services-networking/service/#publishing-services---service-types)であるため、コンテナクラスタ内でのみアクセス可能です。`ClusterIP`はServiceが指し示すPodの集合に対して単一のIPアドレスを提供します。このIPアドレスはクラスタ内でのみアクセス可能です。

Guestbookにアクセスできるゲストを求めるのであれば、フロントエンドServiceを外部から見えるように構成しなければなりません。そうすると、クライアントはコンテナクラスタの外側からServiceへリクエストを送ることができます。Minikubeは`NodePort`を通してのみServiceを公開できます。

{{< note >}}
**メモ:** Google Compute Engine や Google Kubernetes Engineのようないくつかのクラウドプロバイダは外部ロードバランサをサポートします。使用しているクラウドプロバイダがロードバランサをサポートしていて、それを使いたいと考えるのなら、`type: NodePort`をコメントアウトして、`type: LoadBalancer`をアンコメントしてください。
{{< /note >}}

{{< codenew file="application/guestbook/frontend-service.yaml" >}}

1. `frontend-service.yaml`ファイルのフロントエンドServiceを適用する

      ```shell
      kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml
      ```

2. フロントエンドServiceが起動してるか確かめるために、Serviceのリストを問い合わせます。

      ```shell
      kubectl get services
      ```

      出力は次のようになる

      ```
      NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
      frontend       ClusterIP   10.0.0.112   <none>       80:31323/TCP   6s
      kubernetes     ClusterIP   10.0.0.1     <none>        443/TCP        4m
      redis-master   ClusterIP   10.0.0.151   <none>        6379/TCP       2m
      redis-slave    ClusterIP   10.0.0.223   <none>        6379/TCP       1m
      ```

### `NodePort`経由でフロントエンドServiceを見る {#viewing-the-frontend-service-via-nodeport}

このアプリケーションをMinikubeまたはローカルクラスタにデプロイしたのであれば、Guestbookを見るためのIPアドレスを見つける必要があります。

1. フロントエンドServiceのIPアドレスを得るために以下のコマンドを実行する

      ```shell
      minikube service frontend --url
      ```

      出力は次のようになる

      ```
      http://192.168.99.100:31323
      ```

2. IPアドレスをコピーし、ブラウザに貼り付けてGuestbookを見る

### `LoadBalancer`経由でフロントエンドServiceを見る {#viewing-the-frontend-service-via-loadbalancer}

`frontend-service.yaml`マニフェストのServiceタイプを`LoadBalancer`にしてデプロイしたのであれば、Guestbookを見るためのIPアドレスを見つける必要があります。

1. フロントエンドServiceのIPアドレスを得るために以下のコマンドを実行する

      ```shell
      kubectl get service frontend
      ```

      出力は次のようになる

      ```
      NAME       TYPE        CLUSTER-IP      EXTERNAL-IP        PORT(S)        AGE
      frontend   ClusterIP   10.51.242.136   109.197.92.229     80:32372/TCP   1m
      ```
2. 外部IPアドレスをコピーし、ブラウザに貼り付けてGuestbookを見る

## Webフロントエンドのスケール {#scale-the-web-frontend}

サービスがDeploymentコントローラを使うServiceとして定義されているため、スケールアップやダウンは簡単です。

1. フロントエンドPodの数をスケールアップするため、次のコマンドを実行する

      ```shell
      kubectl scale deployment frontend --replicas=5
      ```

2. 実行しているフロントエンドPodの数を確かめるために、Podのリストを問い合わせる

      ```shell
      kubectl get pods
      ```

      出力は次のようになる

      ```
      NAME                            READY     STATUS    RESTARTS   AGE
      frontend-3823415956-70qj5       1/1       Running   0          5s
      frontend-3823415956-dsvc5       1/1       Running   0          54m
      frontend-3823415956-k22zn       1/1       Running   0          54m
      frontend-3823415956-w9gbt       1/1       Running   0          54m
      frontend-3823415956-x2pld       1/1       Running   0          5s
      redis-master-1068406935-3lswp   1/1       Running   0          56m
      redis-slave-2005841000-fpvqc    1/1       Running   0          55m
      redis-slave-2005841000-phfv9    1/1       Running   0          55m
      ```

3. フロントエンドPodの数をスケールダウンするため、次のコマンドを実行する

      ```shell
      kubectl scale deployment frontend --replicas=2
      ```

4. 実行しているフロントエンドPodの数を確かめるために、Podのリストを問い合わせる

      ```shell
      kubectl get pods
      ```

      出力は次のようになる

      ```
      NAME                            READY     STATUS    RESTARTS   AGE
      frontend-3823415956-k22zn       1/1       Running   0          1h
      frontend-3823415956-w9gbt       1/1       Running   0          1h
      redis-master-1068406935-3lswp   1/1       Running   0          1h
      redis-slave-2005841000-fpvqc    1/1       Running   0          1h
      redis-slave-2005841000-phfv9    1/1       Running   0          1h
      ```
        
{{% /capture %}}

{{% capture cleanup %}}
DeploymentやServiceを削除することで、実行しているPodも削除されます。1つのコマンドで複数のリソースを削除するためにはLabelを使います。

1. すべてのPod, Deployment, Serviceを削除するために次のコマンドを実行する

      ```shell
      kubectl delete deployment -l app=redis
      kubectl delete service -l app=redis
      kubectl delete deployment -l app=guestbook
      kubectl delete service -l app=guestbook
      ```

      出力は次のようになる
      
      ```
      deployment.apps "redis-master" deleted
      deployment.apps "redis-slave" deleted
      service "redis-master" deleted
      service "redis-slave" deleted
      deployment.apps "frontend" deleted
      service "frontend" deleted
      ```

2. Podが実行していないことを確かめるために、Podのリストを問い合わせる

      ```shell
      kubectl get pods
      ```

      出力は次のようになる

      ```
      No resources found.
      ```

{{% /capture %}}

{{% capture whatsnext %}}
* [Kubernetesの基礎](/docs/tutorials/kubernetes-basics/)を完了する
* [Persistent Volumes for MySQL and Wordpress](/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/#visit-your-new-wordpress-blog)を使ってブログを作るためにKubernetesを使う
* [アプリケーションへの接続](/docs/concepts/services-networking/connect-applications-service/)についての詳細を参照する
* [リソースの管理](/docs/concepts/cluster-administration/manage-deployment/#using-labels-effectively)についての詳細を参照する
{{% /capture %}}
