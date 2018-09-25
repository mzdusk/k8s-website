---
title: アドミッションコントローラの使用
content_template: templates/concept
weight: 30
---

{{% capture overview %}}
このページではアドミッションコントローラの概要について述べます。
{{% /capture %}}

{{% capture body %}}
## アドミッションコントローラとは何ですか？ {#what-are-they}

アドミッションコントローラはオブジェクトの永続化の前かつリクエストが認証・認可された後にKubernetes APIサーバへのリクエストを横取りするコードの一部です。コントローラは下の[リスト](#what-does-each-admission-controller-do)で構成され、`kube-apiserver`バイナリにコンパイルされ、クラスタ管理者のみが構成できます。このリストには特別なコントローラが2つあります。MutationAdmissionWebhookとValidatingAdmissionWebhookです。これらはそれぞれAPIで構成される[アドミッションコントロールWebhook](/ja/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks)の変化と検証を実行します。

アドミッションコントローラは"検証"と"変化"またはその両方を行うことができます。変化コントローラは許可されたオブジェクトを編集できますが、検証コントローラはできません。

アドミッションコントロールプロセスは2つのフェーズで実行されます。最初のフェーズでは、変化コンロトーラが実行されます。次のフェーズでは、検証コントローラが実行されます。いくつかのコントローラは両方行うことに注意してください。

いずれかのコントローラがどこかのフェーズでリクエストを拒絶すると、リクエスト全体が即座に拒絶され、エンドユーザにエラーが返されます。

最後に、時々問題となるオブジェクトの変化に加え、アドミッションコントローラは時々副作用、すなわち、処理中のリクエストの一部として関連するリソースが変更されてしまう可能性があります。クオータ使用量の増加が、これがなぜ必要なのかを示す規範的な例です。与えられたアドミッションコントローラは与えらえれたリクエストが他のアドミッションコントローラすべてに渡ると確実にはわからないので、このような副作用には類似する再利用処理もしくは調整処理が必要です。

## なぜこれが必要なのですか？ {#why-do-i-need-them}

Kubernetesの高度な機能の多くは機能を正しくサポートするためにアドミッションコントローラを有効にすることが必要になります。その結果、正しいアドミッションコントローラのセットが適切に構成されていないKubernetes APIサーバは不完全なサーバで期待するすべての機能をサポートしないでしょう。

## アドミッションコントローラを有効にするにはどうすればよいですか？ {#how-do-i-turn-on-an-admission-controller}

Kubernetes APIサーバフラグ `enable-admission-plugin`は、クラスタでオブジェクトが編集される前に実行するアドミッションコントロールプラグインのカンマ区切りリストを取ります。例えば、次のコマンドラインは`NamespaceLifecycle`と`LimitRanger`のアドミッションコントロールプラグインを有効にします。

```shell
kube-apiserver --enable-admission-plugins=NamespaceLifecyle,LimitRanger ...
```

{{< note >}}
**メモ:** クラスタをデプロイした方法やどのようにAPIサーバが起動するかによって、異なる方法で設定を適用する必要があるかもしれません。例えば、APIサーバがsystemdサービスによってデプロイされているのであればsystemd unitファイルを編集しなければならないでしょうし、Kubernetesがセルフホストされる方法でデプロイされていれば、APIサーバのマニフェストファイルを編集しなければならないでしょう。
{{< /note >}}

## アドミッションコントローラを無効にするにはどうすればよいですか？ {#how-do-i-turn-off-an-admission-contoller}

Kubernetes APIサーバフラグ `desable-admission-plugins`は、デフォルトで有効になっているプラグインも含め、無効にするアドミッションコントロールプラグインのカンマ区切りリストを取ります。

```shell
kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...
```

## それぞれのアドミッションコントローラは何をするのですか？ {#what-does-each-admission-controller-do}

### AlwaysAdmint (非推奨) {#alwaysadmit}

自身ですべてのリクエストを通過させるためにはこのアドミッションコントローラを使います。AlwaysAdmitは実用的な意味がないので非推奨です。

### AlwaysPullImages {#alwayspullimages}

このアドミッションコントローラはImage Pull PolocyをAlwaysに強制するためにすべての新規Podを編集します。ユーザがプルするための資格情報を持つイメージだけを使えることを保証できるので、マルチテナントクラスタで有用です。このアドミッションコントローラがなければ、一度ノードにイメージをプルすると、あらゆるユーザのあらゆるPodは (Podが正しいノードにスケジュールされると仮定すれば) そのイメージの名前を知っているだけで、そのイメージに対する認可チェックなしに使うことができます。このアドミッションコントローラが有効になると、コンテナが開始する前に常にイメージがプルされるので、有効な資格情報が必要となります。

### AlwaysDeny {#alwaysdeny}

すべてのリクエストを拒絶します。AlwaysDenyは実用的な意味がないので非推奨です。

### DefaultStorageClass {#defaultstorageclass}

このアドミッションコントローラは特定のStorage Classを要求がない`PersistentVolumeClaim`オブジェクトの作成を監視し、それにデフォルトStorage Classを自動で追加します。これにより、特別なStorage Classを要求しないユーザはそれに対して注意を払う必要がなくなり、デフォルトのものを得られます。

このアドミッションコントローラはデフォルトStorage Classは構成されていなければ何もしません。複数のStorage Classがデフォルトであるとマークされていれば、あらゆる`PersistentVolumeClaim`の作成はエラーで拒絶され、管理者は`StorageClass`オブジェクトを再検査しデフォルトを1つにしなければなりません。このアドミッションコントローラは`PersistentVolumeClaim`の更新は無視します。作成の時のみ動作します。

PersistentVolumeClaimとStorage Classについてと、Storage Classをデフォルトとマークする方法については、[PersistentVolume](/ja/docs/concepts/storage/persistent-volumes/)のドキュメントを参照してください。

### DefaultTolerationSeconds {#defaulttolerationseconds}

このアドミッションコントローラは、Podが`node.kubernetes.io/not-ready:NoExecute`または`node.alpha.kubernetes.io/unreachable:NoExecute`の汚染に対する耐性を持っていなければ、5分間`notready:NoExecute`と`unreachable:NoExecute`の汚染に耐えられるように、デフォルトの免除される耐性をPodに設定します。

### DenyExecOnPrivileged (非推奨) {#denyexeconprivileged}

このアドミッションコントローラは、Podが特権コンテナを持っていれば、Podでコマンドを実行するためにすべてのリクエストを横取りします。

クラスタが特権コンテナをサポートしていて、これらのコンテナでコマンドを実行するエンドユーザの能力を制限したいのであれば、このアドミッションコントローラを有効にすることを強く推奨します。

この機能は[DenyEscalatingExec](#denyescalatingexec)に統合されました。

### DenyEscalatingExec {#denyescalatingexec}

このアドミッションコントローラは、ホストアクセスを許可する昇格された特権で実行するPodに対するコマンドの実行と付加を拒否します。これは特権で実行したり、ホストのIPC名前空間へのアクセス権を持っていたり、ホストのPID名前空間へのアクセス権を持っていたりするPodも含まれます。

クラスタが昇格された特権で実行するコンテナをサポートしていて、これらのコンテナでコマンドを実行するエンドユーザの能力を制限したいのであれば、このアドミッションコントローラを有効にすることを強く推奨します。

### EventRateLimit (alpha) {#eventratelimit}

このアドミッションコントローラはAPIサーバがイベントリクエストであふれる問題を軽減します。クラスタ管理者は次の方法でイベントレートリミットを指定できます。

 * 確実にAPIサーバの`--runtime-config`フラグに`eventratelimit.admission.k8s.io/v1alpha1=true`が含まれるようにする
 * `EventRateLimit`アドミッションコントローラを有効にする
 * APIサーバのコマンドラインフラグ `--admission-control-config-file`に提供されるファイルから`EventRateLimit`構成ファイルを参照する。

```yaml
kind: AdmissionConfiguration
apiVersion: apiserver.k8s.io/v1alpha1
plugins:
- name: EventRateLimit
  path: eventconfig.yaml
...
```

構成で指定できるリミットが4タイプあります。

 * `Server`: APIサーバによって受け取られたすべてのイベントリクエストは単一バケットを共有する
 * `Namespace`: 各名前空間が専用のバケットを持つ
 * `User`: 各ユーザにバケットが割り当てられる
 * `SourceAndObject`: バケットはソースとイベントの関連するオブジェクトの組合せによって割り当てられる

以下がこの構成の例の`eventconfig.yaml`です。

```yaml
kind: Configuration
apiVersion: eventratelimit.admission.k8s.io/v1alpha1
limits:
- type: Namespace
  qps: 50
  burst: 100
  cacheSize: 2000
- type: User
  qps: 10
  burst: 50
```

詳細は[EventRateLimitの提案](https://git.k8s.io/community/contributors/design-proposals/api-machinery/admission_control_event_rate_limit.md)を参照してください。

### ExtendedResourceToleration {#extendedresourcetoleration}

このプラグインは拡張リソースの専用ノードを作成することを手助けします。オペレータが拡張リソース (GPUやFPGAなど)の専用ノードを作成したいのであれば、それらはキーとして拡張リソース名で[ノードを汚染](/ja/docs/concepts/configuration/taint-and-toleration/#example-use-cases)していることを期待します。このアドミッションコントローラは、有効であれば、拡張リソースを要求するPodにこのような汚染に対する耐性を自動で付加するので、ユーザは手動でこれらの耐性を付与する必要がありません。

### ImagePolicyWebhook {#imagepolicywebhook}

ImagePolicyWebhookアドミッションコントローラはバックエンドWebhookに合否判定をさせることを許可します。

#### 構成ファイルフォーマット {#configuration-file-format}

ImagePolicyWebhookはバックエンドのふるまいに対するオプションを設定するために構成ファイルを使います。このファイルはjsonまたはyamlで、以下のような形式です。

```yaml
imagePolicy:
  kubeConfigFile: /path/to/kubeconfig/for/backend
  # 認可をキャッシュする秒数
  allowTTL: 50
  # 拒否をキャッシュする秒数
  denyTTL: 50
  # リトライを待つミリ秒数
  retryBackoff: 500
  # Webhookバックエンドが失敗した場合のふるまいを決める
  defaultAllow: true
```

APIサーバのコマンドラインフラグ `--admission-control-config-file` に提供されたファイルからImagePolicyWebhook構成ファイルを参照します。

```yaml
kind: AdmissionConfiguration
apiVersion: apiserver.k8s.io/v1alpha1
plugins:
- name: ImagePolicyWebhook
  path: imagepolicyconfig.yaml
...
```
ImagePolicyWebhook構成ファイルは、バックエンドへの接続をセットアップする[kubeconfig](/ja/docs/concepts/cluster-administration/authenticate-across-clusters-kubeconfig/)でフォーマットされたファイルを参照しなければなりません。

kubeconfigファイルのclusterフィールドはリモートサービスを指さなければならず、ユーザフィールドは返された認可モジュールを含まなければなりません。

```yaml
# clustersはリモートサービスを参照する
clusters:
- name: name-of-remote-imagepolicy-service
  cluster:
    certificate-authority: /path/to/ca.pem    # CA for verifying the remote service.
    server: https://images.example.com/policy # URL of remote service to query. Must use 'https'.

# usersはAPIサーバのWebhook構成を参照する
users:
- name: name-of-api-server
  user:
    client-certificate: /path/to/cert.pem # cert for the webhook admission controller to use
    client-key: /path/to/key.pem          # key matching the cert
```

追加のHTTP構成については[kubeconfig](/docs/concepts/cluster-administration/authenticate-across-clusters-kubeconfig/)のドキュメントを参照してください。

#### リクエストペイロード {#request-payloads}

合否判定に直面すると、APIサーバはアクションを記述した、JSONでシリアル化された`imagepolicy.k8s.io/v1alpha1` `ImageReview`オブジェクトをPOSTします。このオブジェクトは、`*.image-policy.k8s.io/*`にマッチするPodアノテーションと同じように、許可されたコンテナを記述するフィールドを含みます。

Webhook APIオブジェクトは他のKubernetes APIオブジェクトと同じバージョン互換性ルールに従うことに注意してください。実装者はアルファオブジェクトに対する互換性が保証されないことに注意し、確実に正しいデシリアル化をするためにリクエストの"apiVersion"フィールドをチェックすべきです。加えて、APIサーバはimagepolicy.k8s.io/v1alpha1 API拡張グループを有効にしなければなりません (`--runtime-config=imagepolicy.k8s.io/v1alpha1=true`)。

リクエストボディの例:

```
{
  "apiVersion":"imagepolicy.k8s.io/v1alpha1",
  "kind":"ImageReview",
  "spec":{
    "containers":[
      {
        "image":"myrepo/myimage:v1"
      },
      {
        "image":"myrepo/myimage@sha256:beb6bd6a68f114c1dc2ea4b28db81bdf91de202a9014972bec5e4d9171d90ed"
      }
    ],
    "annotations":[
      "mycluster.image-policy.k8s.io/ticket-1234": "break-glass"
    ],
    "namespace":"mynamespace"
  }
}
```

リモートサービスはリクエストのImageReviewStatusフィールドが埋められていて、アクセス許可または不許可のレスポンスがあることを期待します。レスポンスボディの"spec"フィールドは無視されるので省略できます。許可されたレスポンスは次を返します。

```
{
  "apiVersion": "imagepolicy.k8s.io/v1alpha1",
  "kind": "ImageReview",
  "status": {
    "allowed": true
  }
}
```

アクセスを不許可にするためには、サービスは次を返します。

```
{
  "apiVersion": "imagepolicy.k8s.io/v1alpha1",
  "kind": "ImageReview",
  "status": {
    "allowed": false,
    "reason": "image currently blacklisted"
  }
}
```

さらなるドキュメントについては、`imagepolicy.v1alpha1` APIオブジェクトと`plugin/pkg/admission/imagepolicy/admission.go`を参照してください。

#### アノテーションでの拡張 {#extending-with-annotation}

`*.image-policy.k8s.io/*`にマッチするPodのすべてのアノテーションはWebhookに送られます。アノテーションの送信はImage Policyバックエンドを注意しているユーザが追加情報を送り、異なるバックエンド実装に対して異なる情報を受け入れることを許可します。

ユーザがここに置くことのできる情報は以下の通りです。

 * 緊急時にポリシを上書きするためにガラスを割るリクエスト
 * ガラスを割るリクエストを文書化するチケットシステムからのチケット番号
 * 提供されたイメージのイメージIDとして、ポリシサーバにヒントを提供する

どのような場合でも、アノテーションはユーザによって提供され、決してKubernetesによって検証はされません。将来、アノテーションが非常に便利であると決まれば、ImageReviewSpecの名前がついたフィールドに昇格するかもしれません。

### Initializers (alpha) {#initializers}

このアドミッションコントローラは既存の`InitializerConfiguration`に基づいてリソースのイニシャライザを決めます。これは生成されたリソースのメタデータを編集することでイニシャライザを設定します。詳細は[動的アドミッションコントロール](/ja/docs/reference/access-authn-authz/extensible-admission-controllers/)を参照してください。

### LimitPodHardAntiAffinityTopology {#limitpodhardantiaffinitytopology}

このアドミッションコントローラは`requiredDuringSchedulingRequiredDuringExecution`の`kubernetes.io/hostname`以外に`AntiAffinity`トポロジーキーを定義しているPodを拒否します。

### LimitRanger {#limitranger}

このアドミッションコントローラはやってくるリクエストを監視し、`Namespace`の`LimitRange`オブジェクトで列挙された制約に確実に抵触しないようにします。`LimitRange`オブジェクトと使っていれば、これらの制約を強制するためにこのアドミッションコントローラを使わなければなりません。LimitRangerは何も指定しないPodへのデフォルトリソースリクエストを適用するためにも使えます。現在、デフォルトLimitRangerは`default`名前空間のすべてのPodに0.1 CPUの必要条件を適用します。

詳細は[LimitRange設計ドキュメント](https://git.k8s.io/community/contributors/design-proposals/resource-management/admission_control_limit_range.md)と[LimitRangeの例](/ja/docs/tasks/configure-pod-container/limit-range/)を参照してください。

### MutatingAdmissionWebhook (1.9でベータ) {#mutatingadmissionwebhook}

このアドミッションコントローラはリクエストにマッチする変更Webhookを呼び出します。マッチするWebhookは直列に呼ばれます。それぞれがオブジェクトを変更できます。

このアドミッションコントローラは (名前が暗示するように) 変更フェーズでのみ実行します。

これに呼ばれたWebhookが副作用を持つなら (例えばクオータを縮小するなど)、後続のWebhookや検査アドミッションコントローラがリクエストの完了を許可する保証がないので、調整システムを *持たなければなりません*。

MutatingAdmissionWebhookを無効にするのであれば、`--runtime-config`フラグを通じて`admissionregistration.k8s.io/v1beta1`グループ/バージョンの`MutatingWebhookConfiguration`オブジェクトも無効にしなければなりません (バージョン1.9以降では両方とも有効です)。

#### Mutating Webhookを書き、インストールする際の注意点 {#use-caution-when-authoring-and-installing-mutating-webhook}

 * 作成しようとするオブジェクトが違う形で戻ってくると、ユーザは混乱する可能性があります
 * 作成しようとするオブジェクトが違う形で戻ってくると、組込みの制御ループが壊れる可能性があります
   * 元々は設定されていないフィールドを設定することは、元のリクエストで設定されるフィールドを上書きすることよりも、あまり問題になりません。後者を行うことは避けてください
 * これはベータ機能です。将来のKubernetesではこれらのWebhookが行える変更のタイプが制限される可能性があります
 * 組込みリソースやサードパーティリソースに対する制御ループの将来的な変更で今うまく動作するWebhookが壊れる可能性があります。WebhookインストールAPIが確定したとしても、すべてのWebhookのふるまいが将来にわたって保証され、サポートされるわけではありません
 
### NamespaceAutoProvision {#namespaceautoprovision}

このアドミッションコントローラは名前空間が指定されたリソースのすべてのリクエストを検査し、参照する名前空間が存在するかどうかをチェックします。もし見つからなければ名前空間を作成します。このアドミッションコントローラは、名前空間の作成を前もって作成しているものに制限したくないDeploymentに便利です。

### NamespaceExists {#namespaceexists}

このアドミッションコントローラは`Namespace`自身以外で名前空間が指定されたリソースのすべてのリクエストをチェックします。リクエストから参照される名前空間が存在しなければ、そのリクエストは拒絶されます。

### NamespaceLifecycle {#namespacelifecycle}

このアドミッションコントローラは、終了する`Namespace`が新しいオブジェクトを持てないよう強制し、存在しない`Namespace`のリクエストが拒絶されることを保証します。このアドミッションコントローラはシステムが予約している3つの名前空間、`default`, `kube-system`, `kube-public`の削除も防ぎます。

`Namespace`の削除はその名前空間にあるすべてのオブジェクト (PodやServiceなど) を削除する一連のオペレーションを開始します。このプロセスの完全性を強制するため、このアドミッションコントローラを実行することを強く推奨します。

### NodeRestriction {#noderestriction}

このアドミッションコントローラはkubeletが変更できる`Node`と`Pod`オブジェクトを制限します。このアドミッションコントローラによって制限されるため、kubeletは`system:node:<nodeName>`の形のユーザ名のある、`system:nodes`グループの資格情報を使わなければなりません。このkubeletは自身の`Node` APIオブジェクトの編集のみを許され、そのノードに結合した`Pod` APIオブジェクトのみを編集できます。Kubernetes 1.11以降で、kubeletはそれらの`Node` APIオブジェクトの汚染の更新や削除が許されなくなりました。将来のバージョンでは、kubeletが正しく動作するのに必要な最小の権限セットを持つことを保証するために追加の成約が加えられるかもしれません。

### OwnerReferencesPermissionEnforcement {#ownerreferencespermissionenforcement}

このアドミッションコントローラは、オブジェクトの"delete"権限を持つユーザのみが変更できるように、オブジェクトの`metadata.ownerReferences`へのアクセスを保護します。このアドミッションコントローラは、参照された *owner* の`finalizer`サブリソースへの"update"権限を持つユーザのみが変更できるように、オブジェクトの`metadata.ownerReferences[x].blockOwnerDeletion`へのアクセスも保護します。

### PersistentVolumeLabel (非推奨) {#persistentvolumelabel}

このアドミッションコントローラはクラウドプロバイダ (例えばGCEやAWS) によって定義されたPersistentVolumeへリージョンやゾーンのラベルを自動的に付加します。PodとマウントされたPersistentVolumeが同じリージョンやゾーンになることを保証する手助けをします。アドミッションコントローラがPersistentVolumeの自動ラベリングをサポートしなければ、Podが異なるゾーンへボリュームをマウントするのを防ぐために、手動でラベルを追加する必要があるかもしれません。PersistentVolumeLabelは非推奨で、永続ボリュームへのラベリングは[クラウドコントローラマネージャ](/ja/docs/tasks/administer-cluster/running-cloud-controller/)に引き継がれました。1.11から、このアドミッションコントローラはデフォルトで無効です。

### PodNodeSelector {#podnodeselector}

このアドミッションコントローラは、名前空間アノテーションとグローバル構成を読み込んで、名前空間内のどのノードセレクタを使うかをデフォルトで設定したり制限したりします。

#### 構成ファイル形式 {#configuration-file-format}

`PodNodeSelector`はバックエンドのふるまいに対するオプションを設定するために構成ファイルを使います。構成ファイル形式は将来のリリースでバージョン化されたファイルに移動することに注意してください。このファイルは以下の形式を持つjsonまたはyamlです。

```yaml
podNodeSelectorPluginConfig:
 clusterDefaultNodeSelector: <node-selectors-labels>
 namespace1: <node-selectors-labels>
 namespace2: <node-selectors-labels>
```

`PodNodeSelector`構成ファイルをAPIサーバのコマンドラインフラグ `--admission-contorl-config-file` に提供されたファイルから参照します。

```yaml
kind: AdmissionConfiguration
apiVersion: apiserver.k8s.io/v1alpha1
plugins:
- name: PodNodeSelector
  path: podnodeselector.yaml
...
```

#### 構成アノテーション形式 {#configuration-annotation-format}

`PodNodeSelector`はアノテーションキー `scheduler.alpha.kubernetes.io/node-selector`を名前空間にノードセレクタを割り当てるために使います。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/node-selector: <node-selectors-labels>
  name: namespace3
```

#### 内部の動作 {#internal-behavior}

このアドミッションコントローラは次のような動作をします。

1. もし`Namespace`が`scheduler.alpha.kubernetes.io/node-selector`というキーのアノテーションを持つのであれば、ノードセレクタとしてその値を使う
2. 名前空間にそのようなアノテーションがない場合、`PodNodeSelector`プラグイン構成ファイルで定義された`clusterDefaultNodeSelector`をノードセレクタとして使う
3. Podのノードセレクタと名前空間のノードセレクタが衝突しないかどうか評価する。衝突する場合は拒絶される
4. Podのノードセレクタとプラグイン構成ファイルに定義された名前空間固有のホワイトリストが衝突しないかどうか評価する。衝突する場合は拒絶される

{{< note >}}
**メモ:** PodNodeSelectorはラベル付けされたノードで実行するPodを強制することを許可します。PodTolerationRestriction アドミッションプラグインも参照してください。これはPodを汚染されたノードで実行することを防ぐものです。
{{< /note >}}

### PersistentVolumeClaimResize {#persistentvolumeclaimresize}

このアドミッションコントローラは`PersistentVolumeClaim`リサイズリクエストのチェックに対する追加の検証を実行します。

{{< note >}}
**メモ:** ボリュームリサイズについてのサポートはアルファ機能として利用可能です。管理者はリサイズを有効にするため、feature gate `ExpandPersistentVolumes`を`true`に設定しなければなりません。
{{< /note >}}

`ExpandPersistentVolumes` feature gateを有効にした後に、`PersistentVolumeClaimResize`アドミッションコントローラも有効にすることをお勧めします。このアドミッションコントローラは、`allowVolumeExpansion`を`true`に設定することリサイズを明示的に有効にしている`StorageClass`のclaim以外はデフォルトですべてのclaimのリサイズを防ぎます。

例えば、以下の`StorageClass`から作成された`PersistentVolumeClaim`はボリューム拡張をサポートします。

```yaml
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

PersistentVolumeClaimについての詳細は、[PersistentVolumeClaims](/ja/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)を参照してください。

### PodPreset {#podpreset}

このアドミッションコントローラは、マッチするPodPresetで指定されたフィールドをPodに追加します。詳細は[PodPresetのコンセプト](/ja/docs/concepts/workloads/pods/podpreset/)と[PodPresetを使ってPodに情報を追加する](/docs/tasks/inject-data-application/podpreset)を参照してください。

### PodSecurityPolicy {#podsecuritypolicy}

このアドミッションコントローラはPodの作成・変更時に動作し、リクエストされたセキュリティコンテキストと利用可能なPod Security Policyに基づいてPodを受け入れるかどうかを判断します。

Kubernetes 1.6.0未満では、extensions/v1beta1/podsecurity API拡張グループを有効にしなければなりません (`--runtime-config=extensions/v1beta1/podsecuritypolicy=true`)。

詳細は[Pod Security Policyのドキュメント](/docs/concepts/policy/pod-security-policy/)を参照してください。

### PodTolerationRestriction {#podtolerationrestriction}

このアドミッションコントローラは最初にPodの耐性と名前空間の耐性間で衝突を検査し、衝突があればPodリクエストを拒絶します。次に名前空間の耐性をPodの耐性にマージします。マージされた耐性は名前空間のホワイトリストに対してチェックされます。チェックが成功すれば、Podリクエストは受け入れられます。

Podの名前空間が耐性のデフォルトやホワイトリストを持っておらず、クラスタレベルで指定されているものがあれば、それが使われる。

名前空間への耐性は`scheduler.alpha.kubernetes.io/defaultTolerations`と`scheduler.alpha.kubernetes.io/tolerationsWhitelist`を通じて割り当てられる。

### Priority {#priority}

Priorityアドミッションコントローラは`priorityClassName`フィールドを使い、優先度の整数値を追加します。Priority Classが見つかれなければ、そのPodは拒絶されます。

### ResourceQuota {#resourcequota}

このアドミッションコントローラはやってくるリクエストを監視し、`Namespace`の`ResourceQuota`に列挙された制約に違反しないことを保証します。Kubernetesのデプロイで`ResourceQuota`を使っていれば、クオータ制約を強制するためにこのアドミッションコントローラを使わなければなりません。

詳細は[resourceQuota設計ドキュメント](https://git.k8s.io/community/contributors/design-proposals/resource-management/admission_control_resource_quota.md)と[Resource Quotaの例]を参照してください。

### SecurityContextDeny {#securitycontextdeny}

このアドミッションコントローラは特定の昇格[SecurityContext](/ja/docs/user-guide/security-context)を設定しようとするPodを拒否します。クラスタがセキュリティコンテキストの取れる値を制限する[Podセキュリティポリシ](/ja/docs/user-guide/pod-security-policy)を使わないのであれば、これを有効にすべきです。

### ServiceAccount {#serviceaccount}

このアドミッションコントローラは[ServiceAccount](/ja/docs/user-guide/service-accounts)に対する自動化を行います。`ServiceAccount`オブジェクトを使うつもりであれば、このアドミッションコントローラを使うことを強く推奨します。

### Storage Object in Use Protection

`StorageObjectInUseProtection`プラグインは新規に作成されるPersistent Volume Claim (PVC)やPersistent Volume (PV)の`kubernetes.io/pvc-protection`ファイナライザや`kubernetes.io/pv-protection`ファイナライザを追加します。ユーザがPVCまたはPVを削除する場合、Protectionコントローラによってそれぞれのファイナライザが削除されるまで削除されません。詳細は[Storage Object in Use Protection](/ja/docs/concepts/storage/persistent-volumes/#storage-object-in-use-protection)を参照してください。

### ValidatingAdmissionWebhook (1.8ではアルファ、1.9ではベータ) {#validatingadmissionwebhook}

このアドミッションコントローラはリクエストにマッチする検証Webhookを呼び出します。マッチするWebhookは並列に呼び出されます。もしリクエストのいずれかが拒絶されれば、そのリクエストは失敗します。このアドミッションコントローラは検証フェーズでのみ実行します。つまり、`MutatingAdmissionWebhook`アドミッションコントローラから呼ばれるWebhookとは対照的に、ここで呼び出されるWebhookはオブジェクトの変更ができません。

ここで呼び出されるWebhookが (例えばクオータを減らすなどの) 副作用を持つのであれば、後続のWebhookや他の検証アドミッションコントローラがリクエストを拒絶する可能性があるので、調整システムを *持たなければなりません。*

ValidatingAdmissionWebhookを無効にするのであれば、`admissionregistration.k8s.io/v1beta1`グループ/バージョンにある`ValidatingWebhookConfiguration`オブジェクトも、`--runtime-config`フラグを使って無効にしなければなりません (バージョン1.9以降では両方ともデフォルトで有効になっています)。


## お勧めのアドミッションコントローラセットはありますか？

あります。

Kubernetes 1.10以降では、以下のセットを実行することをお勧めします。`--enable-admission-plugins`フラグを使ってください (**順番は関係ありません**)。

{{< note >}}
**Note:** `--admission-control`は1.10で非推奨となり、`--enable-admission-plugins`に置き換えられました。
{{< /note >}}

```shell
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
```

Kubernetes 1.9以前では、以下のセットを実行することをお勧めします。`--admission-control`フラグを使ってください (**順番は重要です**)。

* v1.9

  ```shell
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
  ```
  
  * 1.9でくり返し言及しておく価値があることは、変更フェーズの後に検証フェーズが行われるということです。例えば、`ResourceQuota`は検証フェーズで実行するので、最後に実行するアドミッションコントローラです。`MutatingAdmissionWebhook`は変更フェーズで実行するので、`ResourceQuota`よりも前にいます。
  
    これより前のバージョンでは変更対検証の概念がないので、アドミッションコントローラは指定された順に実行します。

* v1.6 - v1.8

  ```shell
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds
  ```

* v1.4 - v1.5

  ```shell
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
  ```

* v1.2 - v1.3

  ```shell
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota
  ```

* v1.0 - v1.1

  ```shell
  --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,PersistentVolumeLabel,ResourceQuota
  ```
{{% /capture %}}
