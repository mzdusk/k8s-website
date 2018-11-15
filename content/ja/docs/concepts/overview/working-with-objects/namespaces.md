---
title: 名前空間
content_template: templates/concept
weight: 30
---

{{% capture overview %}}

Kubernetesは、同じ物理クラスタ上での複数の仮想クラスタをサポートしています。これらの仮想クラスタは
名前空間と呼ばれます。

{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## いつ複数の名前空間を使うのか {#when-to-use-multiple-namespaces}

名前空間は、複数のチームやプロジェクトにまたがった多くのユーザがいる環境での利用を想定しています。
2、30人程度のユーザのクラスタでは、名前空間を作成したり、名前空間について考えたりする必要はまったく
ありません。名前空間が提供する機能を必要としてから、使い始めてください。

名前空間は名前のスコープを提供します。リソースの名前は名前空間内では一意である必要がありますが、
名前空間にまたがって一意である必要はありません。

名前空間は、複数のユーザでクラスタリソースを ([リソースクオータ](/ja/docs/concepts/policy/resource-quotas/)
経由で) 分割する方法のひとつです。

Kubernetesの将来のバージョンでは、同じ名前空間のオブジェクトはデフォルトで同じアクセス制限ポリシを
持つようになります。

同じソフトウェアの異なるバージョンなど、わずかに異なるリソースを分割するだけであれば、複数の名前空間を
使う必要はありません。同じ名前空間でリソースを区別するには
[ラベル](/docs/concepts/overview/working-with-objects/labels/)を使ってください。

## 名前空間の利用 {#working-with-namespaces}

名前空間の作成と削除は、[名前空間の管理者ガイド](/docs/tasks/administer-cluster/namespaces/)に書かれています。

### 名前空間の閲覧 {#viewing-namespaces}

現在クラスタで使われている名前空間を列挙するには、次のようにします。

```shell
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    1d
kube-system   Active    1d
kube-public   Active    1d
```

Kubernetesは3つの初期名前空間で始まります。

   * `default`: 名前空間を持たないオブジェクトのためのデフォルト名前空間
   * `kube-system`: Kubernetesシステムによって作成されるオブジェクトのための名前空間
   * `kube-public`: この名前空間は自動的に作成され、すべてのユーザ (認証されていないものも含む) から
     読み込み可能です。クラスタ全体でパブリックに閲覧可能で読み込み可能であるべきリソースがある場合に
     備えて、この名前空間はクラスタが使うためにほとんど予約されています。この名前空間のパブリックな
     面は利便性のためだけで、必須ではありません。

### リクエストに対する名前空間の設定 {#setting-the-namespace-for-a-request}

リクエストに対して一時的に名前空間を設定するには、`--namespace`フラグを使います。

例:

```shell
$ kubectl --namespace=<insert-namespace-name-here> run nginx --image=nginx
$ kubectl --namespace=<insert-namespace-name-here> get pods
```

### 名前空間プリファレンスの設定 {#setting-the-namespace-preference}

現在のコンテキストで、以降のすべてのkubectlコマンドに対する名前空間を永続的に保存できます。

```shell
$ kubectl config set-context $(kubectl config current-context) --namespace=<insert-namespace-name-here>
# Validate it
$ kubectl config view | grep namespace:
```

## 名前空間とDNS {#namespaces-and-dns}

[Service](/docs/concepts/services-networking/service/)を作成する時、対応する
[DNSエントリ](/docs/concepts/services-networking/dns-pod-service/)が作成されます。このエントリは
`<service-name>.<namespace-name>.svc.cluster.local`という形になっていて、コンテナが
`<service-name>`のみを使えば、名前空間のローカルのサービスに解決されることを意味しています。
これは、DeploymentやStaging、Productionといった、複数の名前空間で同じ構成を使う際に便利です。
名前空間を横断して到達したい場合は、完全修飾ドメインメイ (FQDN) を使う必要があります。

## すべてのオブジェクトが名前空間にあるわけではない {#not-all-objects-are-in-a-namespace}

ほとんどのKubernetesリソース (Pod、Service、Replicationコントローラなど) は同じ名前空間にあります。
しかしながら、名前空間リソース自身は名前空間に属しません。また、[ノード](/ja/docs/concepts/architecture/nodes/)
やPersistentVolumeのような低レベルリソースは、どの名前空間にも属しません。

名前空間に属すリソースと属さないリソースを見るには、次のコマンドを実行してください。

```shell
# In a namespace
$ kubectl api-resources --namespaced=true

# Not in a namespace
$ kubectl api-resources --namespaced=false
```

{{% /capture %}}

