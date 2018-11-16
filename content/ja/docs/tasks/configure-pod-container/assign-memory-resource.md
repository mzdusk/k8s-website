---
title: コンテナとPodにメモリリソースを割り当てる
content_template: templates/task
weight: 10
---

{{% capture overview %}}

このページでは、メモリ *要求* とメモリ *上限* をコンテナに割り当てる方法を説明します。
コンテナは要求した量のメモリが保証されますが、その上限以上に使うことは許されません。

{{% /capture %}}

{{% capture prerequisites %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

クラスタの各ノードは少なくとも300 MiBのメモリが必要です。

このページのいくつかのステップでは、クラスタで
[metrics-server](https://github.com/kubernetes-incubator/metrics-server)
サービスを実行する必要があります。もし、稼動しているmetrics-serverがなければ、
それらのステップをスキップすることができます。

Minikubeを実行していれば、以下のコマンドでmetrics-serverを有効にできます。

```shell
minikube addons enable metrics-server
```

metrics-serverが稼動しているか、または他のリソースメトリクスAPI (`metrics.k8s.io`) の
プロバイダがあるかどうかを見るには、以下のコマンドを実行します。

```shell
kubectl get apiservices
```

リソースメトリクスAPIが利用可能であれば、出力に`metrics.k8s.io`への参照が含まれます。

```shell
NAME
v1beta1.metrics.k8s.io
```

{{% /capture %}}

{{% capture steps %}}

## 名前空間を作成する {#create-a-namespace}

この演習で作成するリソースがクラスタの他の部分と分離するように、名前空間を作成します。

```shell
kubectl create namespace mem-example
```

## メモリ要求とメモリ上限を指定する {#specify-a-memory-reques-and-a-memory-limit}

コンテナに対してメモリ要求を指定するには、コンテナのリソースマニフェストに`resources:requests`
フィールドを含めます。メモリ上限を指定するには、`resources:limits`を含めます。

この演習では、1つのコンテナを持つPodを作成します。このコンテナは100MiBのメモリ要求と、200MiBの
メモリ上限を持ちます。Podの構成ファイルは次のようになります。

{{< codenew file="pods/resource/memory-request-limit.yaml" >}}

構成ファイルの`args`セクションは、コンテナを起動する時の引数を提供します。`"--vm-bytes", "150M"`
という引数は、コンテナに150 MiBのメモリを割り当てるよう伝えます。

Podを作成します。

```shell
kubectl create -f https://k8s.io/examples/pods/resource/memory-request-limit.yaml --namespace=mem-example
```

Podコンテナが稼動していることを確認します。

```shell
kubectl get pod memory-demo --namespace=mem-example
```

Podの詳細情報を見ます。

```shell
kubectl get pod memory-demo --output=yaml --namespace=mem-example
```

出力はPodの中のコンテナが100 MiBのメモリ要求と200 MiBのメモリ上限を持っていることを示します。

```yaml
...
resources:
  limits:
    memory: 200Mi
  requests:
    memory: 100Mi
...
```

Podのメトリクスを取得するために、`kubectl top`を実行します。

```shell
kubectl top pod memory-demo --namespace=mem-example
```

出力はPodが約162,900,000バイトのメモリを使っていることを示しており、これは約150 MiBになります。
これはPodの100 MiB要求よりも大きいですが、Podの200 MiB上限にはおさまっています。

```
NAME                        CPU(cores)   MEMORY(bytes)
memory-demo                 <something>  162856960
```

Podを削除します。

```shell
kubectl delete pod memory-demo --namespace=mem-example
```

## コンテナのメモリ上限を超える {#exceed-a-containers-memory-limit}

ノードのメモリが利用可能であれば、コンテナはそのメモリ要求を超えることができます。ですがコンテナは
そのメモリ上限を超えて使うことは許されません。コンテナが上限よりも多くメモリを割り当てれば、
コンテナは終了の対象となります。コンテナが上限を超えてメモリを使い続ければ、コンテナは終了されます。
終了したコンテナが再起動できるのであれば、他のタイプのランタイム障害の時と同じようにkubeletはそれを
再起動します。

この演習では、上限を超えたメモリを割り当てようとするPodを作成します。50 MiBのメモリ要求と100 MiBの
メモリ上限のあるコンテナを持つPodの構成ファイルを示します。

{{< codenew file="pods/resource/memory-request-limit-2.yaml" >}}

構成ファイルの`args`セクションで、コンテナが250 MiBのメモリを割り当てようとするのがわかります。
これは上限である100 MiBよりもかなり大きい値です。

Podを作成します。

```shell
kubectl create -f https://k8s.io/examples/pods/resource/memory-request-limit-2.yaml --namespace=mem-example
```

Podの詳細情報を見ます。

```shell
kubectl get pod memory-demo-2 --namespace=mem-example
```

ここで、コンテナは稼動しているか、殺されているかしている可能性があります。コンテナが殺されるまで上の
コマンドをくり返します。

```shell
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          24s
```

コンテナステータスのさらなる詳細を取得します。

```shell
kubectl get pod memory-demo-2 --output=yaml --namespace=mem-example
```

出力はコンテナがメモリ不足 (OOM) によって殺されたことを示します。

```shell
lastState:
   terminated:
     containerID: docker://65183c1877aaec2e8427bc95609cc52677a454b56fcb24340dbd22917c23b10f
     exitCode: 137
     finishedAt: 2017-06-20T20:52:19Z
     reason: OOMKilled
     startedAt: null
```

この演習でのコンテナは再起動ができるので、kubeletはこれを再起動します。コンテナがくり返し殺されて再起動される
様子を見るために、次のコマンドを何度かくり返します。

```shell
kubectl get pod memory-demo-2 --namespace=mem-example
```

出力はコンテナが殺され、再起動され、また殺されをくり返していることを示します。

```
kubectl get pod memory-demo-2 --namespace=mem-example
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          37s
```
```

kubectl get pod memory-demo-2 --namespace=mem-example
NAME            READY     STATUS    RESTARTS   AGE
memory-demo-2   1/1       Running   2          40s
```

Pod履歴の詳細情報を見ます。

```
kubectl describe pod memory-demo-2 --namespace=mem-example
```

出力はコンテナが起動を失敗をくり返していることを示します。

```
... Normal  Created   Created container with id 66a3a20aa7980e61be4922780bf9d24d1a1d8b7395c09861225b0eba1b1f8511
... Warning BackOff   Back-off restarting failed container
```

クラスタのノードについての詳細情報を見ます。

```
kubectl describe nodes
```

出力にはメモリ不足の条件によりコンテナが殺された記録が含まれます。

```
Warning OOMKilling  Memory cgroup out of memory: Kill process 4481 (stress) score 1994 or sacrifice child
```

Podを削除します。

```shell
kubectl delete pod memory-demo-2 --namespace=mem-example
```

## ノードに対して大き過ぎるメモリ要求を指定する {#specify-a-memory-request-that-is-too-big-for-your-nodes}

メモリ要求と上限はコンテナに関連しますが、Podのメモリ要求と上限だと考えると便利です。Podのメモリ要求は
Podに含まれるコンテナのメモリ要求の合計です。同じように、Podのメモリ上限はPodに含まれるコンテナの
メモリ上限の合計になります。

Podスケジューリングはこの要求に基づきます。Podは、そのPodのメモリ要求を満たすのに十分なメモリが利用できるノードで
実行するようスケジュールされます。

この演習では、ノードの容量を大きく超えるメモリ要求を持つPodを作成します。1000 GiBのメモリを要求するコンテナを
持つPodの構成ファイルを示します。このメモリ要求はクラスタ内のどのノードの容量も超えると思われるものです。

{{< codenew file="pods/resource/memory-request-limit-3.yaml" >}}

Podを作成します。

```shell
kubectl create -f https://k8s.io/examples/pods/resource/memory-request-limit-3.yaml --namespace=mem-example
```

Podのステータスを見ます。

```shell
kubectl get pod memory-demo-3 --namespace=mem-example
```

出力はPodのステータスがPENDINGであることを示します。つまり、Podはどのノードにもスケジュールされず、
いつまでもPENDING状態のままになります。

```
kubectl get pod memory-demo-3 --namespace=mem-example
NAME            READY     STATUS    RESTARTS   AGE
memory-demo-3   0/1       Pending   0          25s
```

イベントを含むPodの詳細情報を見ます。

```shell
kubectl describe pod memory-demo-3 --namespace=mem-example
```

出力はノードのメモリ不足のためコンテナがスケジュールできなかったことを示します。

```shell
Events:
  ...  Reason            Message
       ------            -------
  ...  FailedScheduling  No nodes are available that match all of the following predicates:: Insufficient memory (3).
```

## メモリの単位 {#memory-units}

メモリリソースはバイトで計測されます。メモリは普通の数値や、接尾辞 (E, P, T, G, M, K, Ei, Pi, Ti, Gi, Mi, Ki)
を持つ固定小数点数で表現されます。例えば、次の値はおよそ同じものを表現します。

```shell
128974848, 129e6, 129M , 123Mi
```

Podを削除します。

```shell
kubectl delete pod memory-demo-3 --namespace=mem-example
```

## メモリ上限を指定しない場合 {#if-you-do-not-specify-a-memory-limit}

コンテナのメモリ上限を指定しない場合、以下の1つが適用されます。

* コンテナは使用するメモリの上限を持ちません。コンテナは、稼動するノードで利用可能なすべてのメモリを使うことができます。

* デフォルトのメモリ上限が設定されている名前空間で稼動するコンテナであれば、自動的にそのデフォルト上限が適用されます。
  クラスタ管理者は、メモリ上限のデフォルト値を指定するために
  [LimitRange](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#limitrange-v1-core)が
  使えます。

## メモリ要求と上限の動機 {#motivation-for-memory-requests-and-limits}

コンテナに対してメモリ要求と上限を構成することで、ノードで利用可能なメモリリソースを効果的に使うことができます。
Podのメモリ要求を低く保つことで、そのPodにスケジュールされる十分な可能性を与えます。メモリ要求より大きな
メモリ制限を持つことで、以下の2つを達成します。

* Podは、メモリが利用可能なところで、動作のバーストができます。
* バースト中に使えるPodのメモリ量は現実的な量に制限されます。

## クリーンアップ {#clean-up}

名前空間を削除します。これで、このタスクで作成したすべてのPodが削除されます。

```shell
kubectl delete namespace mem-example
```

{{% /capture %}}

{{% capture whatsnext %}}

### アプリケーション開発者向け

* [Assign CPU Resources to Containers and Pods](/docs/tasks/configure-pod-container/assign-cpu-resource/)

* [Configure Quality of Service for Pods](/docs/tasks/configure-pod-container/quality-service-pod/)

### クラスタ管理者向け

* [Configure Default Memory Requests and Limits for a Namespace](/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

* [Configure Default CPU Requests and Limits for a Namespace](/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)

* [Configure Minimum and Maximum Memory Constraints for a Namespace](/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)

* [Configure Minimum and Maximum CPU Constraints for a Namespace](/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)

* [Configure Memory and CPU Quotas for a Namespace](/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)

* [Configure a Pod Quota for a Namespace](/docs/tasks/administer-cluster/manage-resources/quota-pod-namespace/)

* [Configure Quotas for API Objects](/docs/tasks/administer-cluster/quota-api-object/)

{{% /capture %}}
