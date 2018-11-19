---
title: コンテナとPodにCPUリソースを割り当てる
content_template: templates/task
weight: 20
---

{{% capture overview %}}

このページではコンテナにCPU *要求* とCPU *上限* を割り当てる方法を説明します。コンテナは要求した量のCPUを
保証されますが、上限を超えてCPUを使うことは許されません。

{{% /capture %}}

{{% capture prerequisites %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

クラスタの各ノードは最低1CPUが必要です。

このページのいくつかのステップでは、クラスタで
[metrics-server](https://github.com/kubernetes-incubator/metrics-server)
が稼動していることが求められます。metrics-serverを稼動させていなければ、それらのステップはスキップできます。

minikubeを実行している場合、以下のコマンドでmetrics-serverを有効にできます。

```shell
minikube addons enable metrics-server
```

metrics-server (または他のリソースメトリクスAPIプロバイダ) が稼動しているかどうかを見るためには、
次のコマンドを実行します。

```shell
kubectl get apiservices
```

リソースメトリクスAPIが利用可能であれば、出力には`metrics.k8s.io`への参照が含まれます。

```shell
NAME
v1beta1.metrics.k8s.io
```

{{% /capture %}}

{{% capture steps %}}

## 名前空間を作成する {#create-a-namespace}

この演習で作成するリソースをクラスタの他の部分と分離するために名前空間を作成します。

```shell
kubectl create namespace cpu-example
```

## CPU要求とCPU上限を指定する {#specify-a-cpu-requests-and-a-cpu-limit}

コンテナのCPU要求を指定するためには、コンテナリソースマニフェストに`resources:requests`フィールドを含めます。
CPU上限を指定するためには、`resources:limits`を含めます。

この演習では、1つのコンテナを持つPodを作成します。このコンテナは0.5 CPUの要求と1 CPUの上限を持ちます。
このPodの構成ファイルは次のようになります。

{{< codenew file="pods/resource/cpu-request-limit.yaml" >}}

構成ファイルの`args`セクションは、コンテナ起動時の引数を提供します。`-cpus "2"`という引数は、コンテナに
2 CPU使うよう伝えます。

Podを作成します。

```shell
kubectl create -f https://k8s.io/examples/pods/resource/cpu-request-limit.yaml --namespace=cpu-example
```

Podのコンテナが動作しているか確認します。

```shell
kubectl get pod cpu-demo --namespace=cpu-example
```

Podの詳細情報を見ます。

```shell
kubectl get pod cpu-demo --output=yaml --namespace=cpu-example
```

出力は、Podのコンテナが500 ミリCPUの要求と、1 CPUの上限を持っていることを示します。

```yaml
resources:
  limits:
    cpu: "1"
  requests:
    cpu: 500m
```

Podのメトリクスを取得するために`kubectl top`を使います。

```shell
kubectl top pod cpu-demo --namespace=cpu-example
```

出力は、Podが974 ミリCPU使っていることを示します。これはPodの構成ファイルで指定した1 CPUの上限よりもわずかに
少ない値です。

```
NAME                        CPU(cores)   MEMORY(bytes)
cpu-demo                    974m         <something>
```

`-cpu "2"`という設定によって、コンテナは2 CPU使うよう構成しましたが、コンテナは約1 CPUを使うことしか許されません。
コンテナが上限よりも多くのCPUリソースを使おうとするため、コンテナのCPU使用量は調整されます。

{{< note >}}
**メモ:** CPU調整についての別の説明として、ノードが十分に利用可能なCPUリソースを持っていないことが挙げられます。
この演習の必要条件が、各ノードで最低1 CPUか必要であるということを思い出してください。コンテナが1 CPUしか持たない
ノードで実行されれば、コンテナで指定したCPU上限にかかわらず、1 CPUを超えることはできません。
{{< /note >}}

## CPUの単位 {#cpu-units}

CPUリソースは *CPU* 単位で計測されます。Kubernetesでの1 CPUは以下と等価です。

* 1 AWS vCPU
* 1 GCP Core
* 1 Azure vCore
* ハイパースレッディングが有効なIntelプロセッサでの1 ハイパースレッド

分数的な値も可能です。0.5 CPUを要求するコンテナは、1 CPUを要求するコンテナの半分の量が保証されます。ミリを
意味する接尾辞 mを使うことができます。例えば、100m CPUと100 ミリCPU、0.1 CPUはすべて同じです。1m以上の精度は
許されません。

CPUは常に絶対量として要求され、相対量は使えません。0.1は、シングルコアでもデュアルコアでも48コアでも同じCPU量です。

Podを削除します。

```shell
kubectl delete pod cpu-demo --namespace=cpu-example
```

## ノードに対して大きすぎるCPU要求を指定する {#specify-a-cpu-request-that-is-too-big-for-your-node}

CPU要求と上限はコンテナに関連しますが、Podが持つものとして考えると便利です。PodのCPU要求は、そのPodのコンテナの
CPU要求の合計になります。同じように、CPU上限は、コンテナのCPU上限の合計になります。

Podのスケジューリングは要求に基づきます。Podは、そのPodの要求を満たすのに十分なCPUリソースを持つノードにのみ
スケジュールされます。

この演習では、ノードのキャパシティをはるかに上回るCPU要求を持つPodを作成します。1つのコンテナを持つPodの
構成ファイルを以下に示します。コンテナは、クラスタないのあらゆるノードのキャパシティを超えると思われる
100 CPUを要求します。

{{< codenew file="pods/resource/cpu-request-limit-2.yaml" >}}

Podを作成します。

```shell
kubectl create -f https://k8s.io/examples/pods/resource/cpu-request-limit-2.yaml --namespace=cpu-example
```

Podのステータスを見ます。

```shell
kubectl get pod cpu-demo-2 --namespace=cpu-example
```

出力は、PodのステータスがPendingであることを示します。つまり、Podはどのノードにもスケジュールされず、
いつまでもPending状態であり続けます。

```shell
kubectl get pod cpu-demo-2 --namespace=cpu-example
NAME         READY     STATUS    RESTARTS   AGE
cpu-demo-2   0/1       Pending   0          7m
```

Podの詳細情報をイベントも含めて見ます。

```shell
kubectl describe pod cpu-demo-2 --namespace=cpu-example
```

出力は、ノードのCPUリソースが不十分なため、コンテナがスケジュールできないことを示します。

```shell
Events:
  Reason			Message
  ------			-------
  FailedScheduling	No nodes are available that match all of the following predicates:: Insufficient cpu (3).
```

Podを削除します。

```shell
kubectl delete pod cpu-demo-2 --namespace=cpu-example
```

## CPU上限を指定しない場合 {#if-you-do-not-specify-a-cpu-limit}

コンテナのCPU上限を指定しない場合、これらの状態の1つが適用されます。

* コンテナは使えるCPUリソースの上限がなくなります。そのコンテナは、実行するノードの利用可能なCPUリソースを全て使えます。

* コンテナがデフォルトCPU上限が設定された名前空間で動作する場合、そのコンテナには自動的にデフォルト上限が割り当てられます。
  クラスタ管理者は、CPU上限のデフォルト値を指定するために、
  [LimitRange](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#limitrange-v1-core/)
  を使えます。

## CPU要求と上限を設定する動機 {#motivation-for-cpu-requests-and-limits}

クラスタで稼動するコンテナのCPU要求と上限を構成することで、クラスタノードで利用可能なCPUリソースを効率的に
使うことができます。PodのCPU要求を低く保つことで、Podがスケジュールされる可能性があります。CPU要求よりも
大きなCPU上限を持つことで、2つのことを達成できます。

* 利用可能なCPUリソースがあれば、Podの動作をバーストさせられます。
* バースト中のPodのCPUリソース量は合理的な量に制限されます。

## クリーンアップ {#clean-up}

名前空間を削除します。

```shell
kubectl delete namespace cpu-example
```

{{% /capture %}}

{{% capture whatsnext %}}

### アプリケーション開発者向け {#for-app-developers}

* [コンテナとPodにメモリリソースを割り当てる](/ja/docs/tasks/configure-pod-container/assign-memory-resource/)

* [Podのサービス品質を構成する](/docs/tasks/configure-pod-container/quality-service-pod/)

### クラスタ管理者向け {#for-cluster-administrators}

* [名前空間のデフォルトメモリ要求と上限を構成する](/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

* [名前空間のデフォフトCPU要求と上限を構成する](/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)

* [名前空間の最小・最大メモリ制約を構成する](/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)

* [名前空間の最小・最大CPU制約を構成する](/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)

* [名前空間のメモリとCPUのクオータを構成する](/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)

* [名前空間のPodクオータを構成する](/docs/tasks/administer-cluster/manage-resources/quota-pod-namespace/)

* [APIオブジェクトのクオータを構成する](/docs/tasks/administer-cluster/quota-api-object/)

{{% /capture %}}
