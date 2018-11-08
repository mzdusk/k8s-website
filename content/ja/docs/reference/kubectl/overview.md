---
title: kubectlの概要
---

`kubectl`はKubernetesクラスタに対してコマンドを実行するコマンドラインインタフェースです。この概要は`kubectl`文法に
ついて取り上げ、コマンド操作について記述し、一般的な例を提供します。各コマンドについての詳細や、サポートされるフラグと
サブコマンドは、[kubectl](/docs/reference/generated/kubectl/kubectl-commands/)リファレンスドキュメントを参照してください。
インストール方法については[kubectlのインストール](/docs/tasks/kubectl/install/)を参照してください。

## 文法 {#syntax}

`kubectl`コマンドを端末から実行するためには、次の文法を使ってください。

```shell
kubectl [command] [TYPE] [NAME] [flags]
```

`command`, `TYPE`, `NAME`, `flags`は次の通りです。

* `command`: 1つもしくは複数のリソースに実行したい操作を指定します。例: `create`, `get`, `describe`, `delete`

* `TYPE`: [リソースタイプ](#resource-types)を指定します。リソースタイプは大文字小文字を区別せず、単数形、複数形、短縮形で
  指定できます。例えば、次のコマンドは同じ出力になります。

      ```shell
      $ kubectl get pod pod1
      $ kubectl get pods pod1
      $ kubectl get po pod1
      ```

* `NAME`: リソースの名前を指定します。名前は大文字小文字を区別します。名前が省略されると、すべてのリソースの詳細が表示
  されます (例: `$ kubectl get pods`)。複数のリソースへの操作を実行する場合、タイプと名前で各リソースを指定するか、

  1つもしくは複数のリソースを指定できます。

  * タイプと名前でリソースを指定する場合は次のようにします。

    * すべて同じタイプであれば、`TYPE1 name1 name2 name<#>`のようにリソースをグループ化します。<br/>
    例: `$ kubectl get pod example-pod1 example-pod2`

    * 個々に複数のリソースを指定する場合は、`TYPE1/name1 TYPE1/name2 TYPE2/name3 TYPE<#>/name<#>`のように指定します。<br/>
    例: `$ kubectl get pod/example-pod1 replicationcontroller/example-rc1`

  * 1つもしくは複数のファイルを指定する場合は、`-f file1 -f file2 -f file<#>`のようにします。

    * YAMLは、構成ファイルについては特にユーザフレンドリなので、[JSONではなくYAMLを使ってください](/docs/concepts/configuration/overview/#general-config-tips)。<br/>
    例: `$ kubectl get pod -f ./pod.yaml`

* `flags`: オプションのフラグを指定します。例えば、`-s`や`--server`フラグはKubernetes APIサーバのアドレスとポートを
  指定するために使います。<br/>
  **重要**: コマンドラインから指定したフラグは、デフォルト値と対応する環境変数を上書きします。

ヘルプが必要であれば、`kubectl help`を実行してください。

## 操作 {#operations}

以下の表に、すべての`kubectl`操作の短い説明と一般的な文法を示します。

Operation       | Syntax    |       Description
-------------------- | -------------------- | --------------------
`annotate`    | `kubectl annotate (-f FILENAME \| TYPE NAME \| TYPE/NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--overwrite] [--all] [--resource-version=version] [flags]` | 1つもしくは複数のリソースに、アノテーションを追加または更新します。
`api-versions`    | `kubectl api-versions [flags]` | 利用可能なAPIバージョンを列挙します。
`apply`            | `kubectl apply -f FILENAME [flags]`| ファイルまたは標準入力から、リソースの構成変更を適用します。
`attach`        | `kubectl attach POD -c CONTAINER [-i] [-t] [flags]` | 出力ストリームを見たり、コンテナとやりとりするために、稼動中のコンテナにアタッチします。
`autoscale`    | `kubectl autoscale (-f FILENAME \| TYPE NAME \| TYPE/NAME) [--min=MINPODS] --max=MAXPODS [--cpu-percent=CPU] [flags]` | Replicationコントローラによって管理されているPodを自動的にスケールします。
`cluster-info`    | `kubectl cluster-info [flags]` | マスタとサービスのエンドポイント情報を表示します。
`config`        | `kubectl config SUBCOMMAND [flags]` | kubeconfigファイルを修正します。詳細は個々のサブコマンドを見てください。
`create`        | `kubectl create -f FILENAME [flags]` | ファイルや標準入力から、1つもしくは複数のリソースを作成します。
`delete`        | `kubectl delete (-f FILENAME \| TYPE [NAME \| /NAME \| -l label \| --all]) [flags]` | ファイルや標準入力、ラベルセレクタや名前、リソースセレクタの指定によってリソースを削除します。
`describe`    | `kubectl describe (-f FILENAME \| TYPE [NAME_PREFIX \| /NAME \| -l label]) [flags]` | 1つもしくは複数のリソースの詳細な状態を表示します。
`edit`        | `kubectl edit (-f FILENAME \| TYPE NAME \| TYPE/NAME) [flags]` | デフォルトエディタを使って、サーバにある1つもしくは複数のリソースの定義を編集し更新します。
`exec`        | `kubectl exec POD [-c CONTAINER] [-i] [-t] [flags] [-- COMMAND [args...]]` | Podのコンテナに対してコマンドを実行します。
`explain`    | `kubectl explain [--include-extended-apis=true] [--recursive=false] [flags]` | さまざまなリソースのドキュメントを取得します。例: pods, nodes, servicesなど。
`expose`        | `kubectl expose (-f FILENAME \| TYPE NAME \| TYPE/NAME) [--port=port] [--protocol=TCP\|UDP] [--target-port=number-or-name] [--name=name] [----external-ip=external-ip-of-service] [--type=type] [flags]` | ReplicationコントローラやService、PodをKubernetes Serviceとして公開します。
`get`        | `kubectl get (-f FILENAME \| TYPE [NAME \| /NAME \| -l label]) [--watch] [--sort-by=FIELD] [[-o \| --output]=OUTPUT_FORMAT] [flags]` | 1つもしくは複数のリソースを列挙します。
`label`        | `kubectl label (-f FILENAME \| TYPE NAME \| TYPE/NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--overwrite] [--all] [--resource-version=version] [flags]` | 1つもしくは複数のリソースのラベルを追加または更新します。
`logs`        | `kubectl logs POD [-c CONTAINER] [--follow] [flags]` | Podのコンテナのログを表示します。
`patch`        | `kubectl patch (-f FILENAME \| TYPE NAME \| TYPE/NAME) --patch PATCH [flags]` | 1つもしくは複数のリソースにパッチを適用します。
`port-forward`    | `kubectl port-forward POD [LOCAL_PORT:]REMOTE_PORT [...[LOCAL_PORT_N:]REMOTE_PORT_N] [flags]` | 1つもしくは複数のPodのポートを転送します。
`proxy`        | `kubectl proxy [--port=PORT] [--www=static-dir] [--www-prefix=prefix] [--api-prefix=prefix] [flags]` | Kubernetes APIサーバへのプロキシを起動します。
`replace`        | `kubectl replace -f FILENAME` | ファイルまたは標準入力からリソースを入れ換えます。
`rolling-update`    | `kubectl rolling-update OLD_CONTROLLER_NAME ([NEW_CONTROLLER_NAME] --image=NEW_CONTAINER_IMAGE \| -f NEW_CONTROLLER_SPEC) [flags]` | 指定したReplicationコントローラとそのPodを徐々に置き換えることで、ローリングアップデートを実行します。
`run`        | `kubectl run NAME --image=image [--env="key=value"] [--port=port] [--replicas=replicas] [--dry-run=bool] [--overrides=inline-json] [flags]` | 指定したイメージをクラスタで実行します。
`scale`        | `kubectl scale (-f FILENAME \| TYPE NAME \| TYPE/NAME) --replicas=COUNT [--resource-version=version] [--current-replicas=count] [flags]` | 指定したReplicationコントローラのサイズを更新します。
`stop`        | `kubectl stop` | 非推奨: 代わりに`kubectl delete`を見てください。
`version`        | `kubectl version [--client] [flags]` | クライアントとサーバのKubernetesバージョンを表示します。

コマンド操作の詳細については、[kubectl](/docs/user-guide/kubectl/)リファレンスドキュメントを参照してください。

## リソースタイプ {#resource-types}

次の表に、サポートされるすべてのリソースタイプとその省略形を列挙します。

リソースタイプ    | 省略形
-------------------- | --------------------
`apiservices` |
`certificatesigningrequests` |`csr`
`clusters` |
`clusterrolebindings` |
`clusterroles` |
`componentstatuses` |`cs`
`configmaps` |`cm`
`controllerrevisions` |
`cronjobs` |
`customresourcedefinition` |`crd`
`daemonsets` |`ds`
`deployments` |`deploy`
`endpoints` |`ep`
`events` |`ev`
`horizontalpodautoscalers` |`hpa`
`ingresses` |`ing`
`jobs` |
`limitranges` |`limits`
`namespaces` |`ns`
`networkpolicies` |`netpol`
`nodes` |`no`
`persistentvolumeclaims` |`pvc`
`persistentvolumes` |`pv`
`poddisruptionbudget` |`pdb`
`podpreset` |
`pods` |`po`
`podsecuritypolicies` |`psp`
`podtemplates` |
`replicasets` |`rs`
`replicationcontrollers` |`rc`
`resourcequotas` |`quota`
`rolebindings` |
`roles` |
`secrets` |
`serviceaccounts` |`sa`
`services` |`svc`
`statefulsets` |
`storageclasses` |

## 出力オプション {#output-options}

あるコマンドの出力をフォーマットしたりソートしたりする方法について、次のセクションで示します。どのコマンドがさまざまな
出力オプションをサポートするかの詳細は、[kubectl](/docs/user-guide/kubectl/)リファレンスドキュメントを参照してください。

### 出力フォーマット {#formatting-output}

すべての`kubectl`コマンドのデフォルト出力形式は、人間に読みやすいプレーンテキスト形式です。特定の形式で詳細を出力する
ために、`-o`または`--output`フラグを、サポートされる`kubectl`コマンドに追加できます。

#### 文法 {#syntax}

```shell
kubectl [command] [TYPE] [NAME] -o=<output_format>
```

`kubectl`操作によって、次の出力形式がサポートされます。

出力形式 | 説明
--------------| -----------
`-o=custom-columns=<spec>` | カンマで区切られた[カスタム列](#custom-columns)の表を表示します。
`-o=custom-columns-file=<filename>` | `<filename>`ファイルの[カスタム列](#custom-columns)テンプレートを使った表を表示します。
`-o=json`     | JSON形式のAPIオブジェクトを出力します。
`-o=jsonpath=<template>` | [jsonpath](/docs/reference/kubectl/jsonpath/)表現で定義されたフィールドを表示します。
`-o=jsonpath-file=<filename>` | `<filename>`ファイルの[jsonpath](/docs/reference/kubectl/jsonpath/)表現で定義されたフィールドを表示します。
`-o=name`     | リソース名のみを表示します。
`-o=wide`     | プレーンテキスト形式に追加情報を付加して出力します。Podの場合、ノード名が含まれます。
`-o=yaml`     | YAML形式のAPIオブジェクトを出力します。

##### 例 {#example}

この例では、次のコマンドはYAML形式のオブジェクトとして、単一Podの詳細を出力します。

```shell
$ kubectl get pod web-pod-13je7 -o=yaml
```

コマンドごとに、どの出力形式がサポートされているのかについては、[kubectl](/docs/user-guide/kubectl/)リファレンスドキュメントを
参照してください。

#### カスタム列 {#custom-columns}

カスタム列を定義し、表に含めたい詳細のみを出力するには、`custom-columns`オプションが使えます。カスタム列は、インラインまたは
テンプレートファイルを使って定義できます。例: `-o=custom-columns=<spec>`または`-o=custom-columns-file=<filename>`

##### 例 {#example}

インライン:

```shell
$ kubectl get pods <pod-name> -o=custom-columns=NAME:.metadata.name,RSRC:.metadata.resourceVersion
```

テンプレートファイル:

```shell
$ kubectl get pods <pod-name> -o=custom-columns-file=template.txt
```

`template.txt`は次のようなファイルです。

```
NAME          RSRC
metadata.name metadata.resourceVersion
```

コマンドを実行した結果:

```shell
NAME           RSRC
submit-queue   610995
```

#### サーバサイド列 {#server-side-columns}

`kubectl`は、オブジェクトについてサーバから特定の列情報を受け取ることをサポートしています。これは与えられた
リソースに対して、サーバがそのリソースに関連する列と行を返すことを意味します。サーバが出力の詳細をカプセル化
することで、同じクラスタに対して使うクライアント全体で、人間に読みやすい出力の一貫性を保てます。

この機能は`kubectl` 1.11からデフォルトで有効になっています。無効にするためには、`kubectl get`コマンドで
`--server-print=false`フラグを追加してください。

##### 例 {#examples}

Podのステータスについての情報を表示するために、以下のようなコマンドを使います。

```shell
kubectl get pods <pod-name> --server-print=false
```

出力は次にようになります。

```shell
NAME       READY     STATUS              RESTARTS   AGE
pod-name   1/1       Running             0          1m
```

### オブジェクトリストのソート {#sorting-list-objects}

ソートされたオブジェクトのリストを出力するためには、`--sort-by`フラグを、サポートする`kubectl`コマンドに追加します。
`--sort-by`フラグで指定された数値や文字列フィールドでオブジェクトがソートされます。フィールドを指定するためには
[jsonpath](/docs/reference/kubectl/jsonpath/)表現を使います。

#### 文法 {#syntax}

```shell
kubectl [command] [TYPE] [NAME] --sort-by=<jsonpath_exp>
```

##### 例 {#example}

名前でソートされたPodのリストを表示するには、次を実行します。

```shell
$ kubectl get pods --sort-by=.metadata.name
```

## 例: 一般的な操作 {#exaples-common-operations}

よく使われる`kubectl`に詳しくなるために、以下の例を役立ててください。

`kubectl create` - ファイルや標準入力からリソースを作成します。

```shell
// example-service.yamlの定義を使ってServiceを作成します
$ kubectl create -f example-service.yaml

// example-controller.yamlの定義を使ってReplicationコントローラを作成します
$ kubectl create -f example-controller.yaml

// <directory>ディレクトリにある、すべての.yaml, .yml,
// .jsonで定義されたオブジェクトを作成します
$ kubectl create -f <directory>
```

`kubectl get` - 1つもしくは複数のリソースを列挙します。

```shell
// プレーンテキスト形式で、すべてのPodを列挙します
$ kubectl get pods

// 追加情報 (ノード名など) も含めて、プレーンテキスト形式ですべてのPodを列挙します
$ kubectl get pods -o wide

// プレーンテキスト形式で、指定された名前のReplicationコントローラを列挙します。
// Tip: 'replicationcontroller'というリソースは'rc'と省略することができます
$ kubectl get replicationcontroller <rc-name>

// プレーンテキスト形式で、すべてのReplicationコントローラとServiceを一緒に列挙します
$ kubectl get rc,services

// プレーンテキスト形式で、すべてのDaemonSetを、初期化ができていないものも含めて列挙します
$ kubectl get ds --include-uninitialized

// server01というノードで実行しているすべてのPodを列挙します
$ kubectl get pods --field-selector=spec.nodeName=server01

// 出力の詳細はサーバに移譲にて、プレインテキスト形式ですべてのPodを列挙します
$ kubectl get pods --experimental-server-print
```

`kubectl describe` - 1つもしくは複数のリソースの詳細な状態を、デフォルトでは初期化されていないものも含めて表示します。

```shell
// <node-name>という名前のノードの詳細を表示します
$ kubectl describe nodes <node-name>

// <pod-name>という名前のPodの詳細を表示します
$ kubectl describe pods/<pod-name>

// <rc-name>という名前のReplicationコントローラによって管理されている、
// すべてのPodの詳細を表示しますReplicationControllerによて作成された
// すべてのPodは、Replicationコントローラの名前が接頭辞になっています
$ kubectl describe pods <rc-name>

// 初期化されていないものは除いて、すべてのPodの詳細を表示します
$ kubectl describe pods --include-uninitialized=false
```

{{< note >}}
**メモ:** `kubectl get`コマンドは、通常同じリソースタイプのリソースを取得するために使われます。
`-o`または`--output`フラグを使って出力形式をカスタマイズできるなど、豊富なフラグを備えていることが
特徴です。特定のオブジェクトの更新を監視するために`-w`または`--watch`フラグを指定できます。
`kubectl describe`コマンドは指定したリソースの詳細を見ることに焦点をあてています。ユーザへのビューを
生成するために、APIサーバに対して多くのAPI呼び出しを実行します。例えば、`kubectl describe node`コマンドは、
ノードについての情報だけでなく、そこで稼動しているPodの概要や、ノードに対して生成されたイベントも取得します。
{{</ note >}}

`kubectl delete` - ファイルや標準入力、ラベルセレクタや名前、リソースセレクタの指定によって、リソースを削除します。

```shell
// pod.yamlファイルで指定したタイプと名前を使って、Podを削除します
$ kubectl delete -f pod.yaml

// name=<label-name>というラベルを持つ、すべてのPodとServiceを削除します
$ kubectl delete pods,services -l name=<label-name>

// name=<label-name>というラベルを持つ、すべてのPodとServiceを、
// 初期化されていないものも含めて削除します
$ kubectl delete pods,services -l name=<label-name> --include-uninitialized

// 初期化されていないものも含めて、すべてのPodを削除します
$ kubectl delete pods --all
```

`kubectl exec` - Podのコンテナに対してコマンドを実行します。

```shell
// <pod-name>というPodで'date'を実行して、出力を取得します。
// デフォルトでは、出力は最初のコンテナからのものになります
$ kubectl exec <pod-name> date

// <pod-name>というPodの<container-name>というコンテナで'date'を実行して、出力を取得します
$ kubectl exec <pod-name> -c <container-name> date

// <pod-name>というPodからインタラクティブなTTYを取得し、/bin/bashを実行します
// デフォルトでは、出力は最初のコンテナからのものになります
$ kubectl exec -ti <pod-name> /bin/bash
```

`kubectl logs` - Podのコンテナのログを表示します。

```shell
// <pod-name>というPodから、ログのスナップショットを返します
$ kubectl logs <pod-name>

// <pod-name>というPodから、ログのストリーミングを開始します。
// これは'tail -f'というLinuxコマンドと同様です
$ kubectl logs -f <pod-name>
```

## 例: プラグインの作成と利用

`kubectl`プラグインの作成と使用に詳しくなる手助けとして、以下の例を使ってください。

```shell
// "kubectl-"で始まる実行ファイル名であれば、どのような言語や名前でもプラグインが作れます
$ cat ./kubectl-hello
#!/bin/bash

# このプラグインは"hello world"という語を出力します
echo "hello world"

// プラグインを書いたら、実行可能にしましょう
$ sudo chmod +x ./kubectl-hello

// そして、PATHの場所に移動させます
$ sudo mv ./kubectl-hello /usr/local/bin

// これで、kubectlプラグインの作成と"インストール"ができました
// これで、あたかも通常のコマンドのように、kubectlからプラグインを使い始めることができます
$ kubectl hello
hello world

// 単純にPATHから削除するだけで、プラグインの"アンインストール"ができます
$ sudo rm /usr/local/bin/kubectl-hello
```

`kubectl`で利用可能なすべてのプラグインを見るために、`kubectl plugin list`サブコマンドを使うことができます。

```shell
$ kubectl plugin list
The following kubectl-compatible plugins are available:

/usr/local/bin/kubectl-hello
/usr/local/bin/kubectl-foo
/usr/local/bin/kubectl-bar

// このコマンドは、プラグインについて、実行可能でない場合や、他のプラグインに上書き
// されてしまう場合についての警告も行います
$ sudo chmod -x /usr/local/bin/kubectl-foo
$ kubectl plugin list
The following kubectl-compatible plugins are available:

/usr/local/bin/kubectl-hello
/usr/local/bin/kubectl-foo
  - warning: /usr/local/bin/kubectl-foo identified as a plugin, but it is not executable
/usr/local/bin/kubectl-bar

error: one plugin warning was found
```

プラグインは、既存のkubectlコマンドを使って、より複雑な機能を構築するための手段と考えることができます。

```shell
$ cat ./kubectl-whoami
#!/bin/bash

# このプラグインは、現在選択されているコンテキストに基づいて、
# 現在のユーザの情報を出力するために、`kubectl config`コマンドを使います
kubectl config view --template='{{ range .contexts }}{{ if eq .name "'$(kubectl config current-context)'" }}Current user: {{ .context.user }}{{ end }}{{ end }}'
```

上のプラグインを実行すると、KUBECONFIGファイルで現在選択されているコンテキストに対するユーザを含む出力が得られます。

```shell
// ファイルを実行可能にします
$ sudo chmod +x ./kubectl-whoami

// PATHに移動します
$ sudo mv ./kubectl-whoami /usr/local/bin

$ kubectl whoami
Current user: plugins-user
```

プラグインのさらなる詳細については、[example cli plugin](https://github.com/kubernetes/sample-cli-plugin)を参照してください。

## 次のステップ {#next-step}

[kubectl](/docs/reference/generated/kubectl/kubectl-commands/)コマンドを使い始めましょう。
