---
title: RBAC認可の使用
content_template: templates/concept
weight: 70
---

{{% capture overview %}}
ロールベースアクセス制御 (Role-Based Access Control; RBAC) は、企業内での個々のユーザのロールに
基づいて計算資源やネットワークリソースへのアクセスを制御する方法です。
{{% /capture %}}

{{% capture body %}}
`RBAC`は認可判定を実行するために`rbac.authorization.k8s.io` APIグループを使い、管理者に
Kubernetes APIを通じた動的なポリシーの構成を可能にします。

1.8からRBACモードは安定になり、rbac.authorization.k8s.io/v1 APIの配下になります。

RBACを有効にするには、`--authorization-mode=RBAC`を指定してapiserverを起動します。

## APIの概要 {#api-overview}

RBAC APIはこのセクションで言及する4つの最上位タイプを定義します。ユーザは他のAPIリソースと同じように
(`kubectl`やAPI呼び出しなどを通じて) これらのリソースとやりとりします。例えば、
`kubectl create -f (resource).yml`がこれらの例で使えますが、理解したい読者は最初に自力でこの
セクションを概観すべきです。

### RoleとClusterRole {#role-and-clusterrole}

RBAC APIでは、権限のセットを表現するルールもロールに含まれます。権限は純粋に追加されるものです
("拒否"するルールはありません)。ロールは`Role`で名前空間に、`ClusterRole`でクラスタレベルに定義できます。

`Role`は単一の名前空間内のリソースへのアクセスを許可するために使われます。"default"名前空間でPodの
参照権限を与える`Role`の例を示します。

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

`ClusterRole`は`Role`と同じ権限を与えることができますが、クラスタレベルなので、以下のアクセス権限も
与えることができます。

* クラスタレベルのリソース (ノードなど)
* 非リソースエンドポイント ("/healthz"など)
* すべての名前空間を横断する名前空間つきリソース (例えば、`kubectl get pods --all-namespaces`を
  実行するときに必要)

以下の`ClusterRole`はすべての指定された名前空間のSecretまたはすべての名前空間を横断するSecretの
参照権限を与えるために使われます (どのように[結合](#rolebinding-and-clusterrolebinding)するかによります)。

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # ClusterRoleは名前空間が付かないので、"namespace"は省略されます
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### RoleBindingとClusterRoleBinding {#rolebinding-and-clusterrolebinding}

ロール結合 (Role Binding) はユーザまたはユーザのセットで定義された権限を与えます。これは対象 (ユーザ、
グループ、サービスアカウント) のリストと許可するロールへの参照を持ちます。`RoleBinding`は同じ
名前空間の`Role`を参照できます。以下の`RoleBinding`は"default"名前空間の"pod-reader"ロールの権限を
ユーザ"jane"に与えます。これは"jane"に"default"名前空間のPodの参照を許可します。

`roleRef`は結合を実際に作る方法です。`kind`は`Role`または`ClusterRole`で、`name`は特定の`Role`
または`ClusterRole`の名前を参照します。以下の例では、このRoleBindingが`roleRef`を使って、
ユーザ"jane"を`pod-reader`という上で作った`Role`に結合します。

```yaml
# このロール結合は"jane"に"default"名前空間のPodの参照を許可します
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane # 名前は大文字小文字を区別します
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # これは結合したいRoleまたはClusterRoleの名前にマッチしなければなりません
  apiGroup: rbac.authorization.k8s.io
```

`RoleBinding`は`RoleBinding`の名前空間内にある`ClusterRole`で定義された名前空間付きリソースへの
権限を与えるために`ClusterRole`を参照することもできます。これにより管理者がクラスタ全体に対する共通の
ロールを定義できるようにし、複数の名前空間で再利用できるようになります。

例えば、以下の`RoleBinding`は`ClusterRole`を参照していますが、"dave" (対象者、大文字小文字が
区別されます) は"development"名前空間 (`RoleBinding`の名前空間) のSecretのみを参照できます。

```yaml
# このロール結合は"dave"に"development"名前空間のSecretの参照を許可します
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets
  namespace: development # これは"development"名前空間内での権限のみを与えます
subjects:
- kind: User
  name: dave # 名前は大文字小文字を区別します
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

最後に、`ClusterRoleBinding`はクラスタレベルの全名前空間で権限を与えるために使うことができます。
以下の`ClusterRoleBinding`は"manager"グループのあらゆるユーザにあらゆる名前空間のSecretを
参照できるようにします。

```yaml
# このクラスタロール結合は"manager"グループの全員にあらゆる名前空間のSecretを参照できるようにします
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # 名前は大文字小文字を区別します
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### リソースへの参照 {#referring-to-resources}

ほとんどのリソースは関連するAPIエンドポイントのURLに現れる"pods"のような名前の文字列表現で表現されます。
しかしながら、Kubernetes APIにはPodに対するログのような"サブリソース"を含むものもあります。Podの
ログエンドポイントのURLは次のようになります。

```http
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

この場合、"pods"は名前空間つきリソースで、"log"はPodのサブリソースです。これをRBACロールで表現するには、
リソースとサブリソースをスラッシュで区切ります。対象にPodとPodのログの両方の参照を許可するためには、
次のように書きます。

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

リソースは`resourceNames`リストを通じて、特定のリクエストに対する名前によって参照することもできます。
指定された場合、"get", "delete", "update", "patch"メソッドを使うリクエストはリソースの個々のインスタンスに
制限されます。対象を単一のconfigmapの"get"と"update"に制限するには、次のように書きます。

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-configmap"]
  verbs: ["update", "get"]
```

特に、`resourceNames`が設定されていると、verbにlist, watch, create, deletecollectionを指定しては
いけません。create, list, watch, deletecollection APIリソースのURLにはリソース名が現れないので、ルールの
`resourceNames`の部分がリクエストにマッチせず、これらのverbsは許可されません。

### 集約されたClusterRole {#aggregated-clusterroles}

1.9から、ClusterRoleは`aggregationRule`を用いて他のClusterRoleと組み合わせて作ることができます。
集約されたClusterRoleの権限はコントローラが管理し、指定されたラベルセレクタにマッチするClusterRoleの
ルールの和集合になります。集約されたClusterRoleの例を示します。

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # ルールはコントローラマネージャによって自動的に設定されます
```

ラベルセレクタにマッチするClusterRoleの作成によって、集約されたClusterRoleにルールが追加されます。
この場合、ラベル`rbac.example.com/aggregate-to-motinoring: true`を持つ他のClusterRoleを
作成することで、ルールが"monitoring" ClusterRoleに追加されます。

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# これらのルールが"monitoring"ロールに追加されます。
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
```

デフォルトのユーザ用ロール (以下で説明します) はClusterRole集約を使います。これにより、管理者が
CustomResourceDefinitionやAggregated APIサーバによって提供されるようなカスタムリソースに対する
ルールをデフォルトのロールに含められるようになります。

例えば、以下のClusterRoleは"admin"と"edit"ロールにカスタムリソースである"CronTabs"を管理させ、
"view"ロールはそのリソースへの参照のみのアクションを実行できるようにします。

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-cron-tabs-edit
  labels:
    # これらの権限を"admin"と"edit"ロールに与えます
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-cron-tabs-view
  labels:
    # これらの権限を"view"ロールに与えます
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch"]
```

### ロールの例 {#role-example}

以下の例では`rules`セクションのみを示します。

コアAPIグループのリソースである"pods"の参照を許可します。

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

"extensions"と"apps" APIグループの両方の"deployments"への参照と書き込みを許可します。

```yaml
rules:
- apiGroups: ["extensions", "apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

"pods"の参照と"jobs"の参照と書き込みを許可します。

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch", "extensions"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

"my-config"という名前の`ConfigMap`の参照を許可します (単一の名前空間にある単一の`ConfigMap`に
制限するために`RoleBingind`で結合させなければなりません)。

```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

コアグループのリソースである"nodes"の参照を許可します (`Node`はクラスタレベルなので、効力を発するには
`ClusterRoleBinding`を使って`ClusterRole`に結合しなければなりません)。

```yaml
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

非リソースエンドポイントである"/healthz"とそのサブパスに"GET"と"POST"のリクエストを許可します
(効力を発するには`ClusterRoleBinding`を使って`ClusterRole`に結合しなければなりません)。

```yaml
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"] # '*' in a nonResourceURL is a suffix glob match
  verbs: ["get", "post"]
```

### 対象への参照 {#referring-to-subjects}

`RoleBinding`や`ClusterRoleBinding`はロールを *対象* に結合します。対象は、グループやユーザ、
サービスアカウントです。

ユーザは文字列で表現されます。これには"alice"のような通常のユーザ名や、"bob@example.com"のような
メールアドレススタイルの名前、文字列として表現した数値のIDがあります。これはKubernetes管理者がユーザ名を
生成するために構成した[認証モジュール](/ja/docs/reference/access-authn-authz/authentication/)によります。
RBAC認可システムは特定の形式を必要としません。しかしながら、`system:`という接頭辞はKubernetesシステムが
使うために予約されているので、管理者は間違ってこの接頭辞がユーザ名に含まれてしまわないようにすべきです。

Kubernetesでのグループ情報は現在認証モジュールによって提供されています。グループはユーザのように文字列で
表現され、その文字列はどのような形式でもかまいませんが、`system:`という接頭辞は予約されています。

[サービスアカウント](/ja/docs/tasks/configure-pod-container/configure-service-account/)は
`system:serviceaccount:`という接頭辞のユーザ名を持ち、`system:serviceaccounts:`という接頭辞の
グループに所属します。

#### ロール結合の例 {#role-binding-examples}

以下の例では`RoleBinding`の`subjects`セクションのみを示します。

"alice@example.com"というユーザに対しては次のようになります。

```yaml
subjects:
- kind: User
  name: "alice@example.com"
  apiGroup: rbac.authorization.k8s.io
```

"frontend-admins"というグループに対しては次のようになります。

```yaml
subjects:
- kind: Group
  name: "frontend-admins"
  apiGroup: rbac.authorization.k8s.io
```

kube-system名前空間のdefaultサービスアカウントに対しては次のようになります。

```yaml
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
```

"qa"名前空間のすべてのサービスアカウントに対しては次のようになります。

```yaml
subjects:
- kind: Group
  name: system:serviceaccounts:qa
  apiGroup: rbac.authorization.k8s.io
```

すべての名前空間のすべてのサービスアカウントに対しては次のようになります。

```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

すべての認証済みユーザに対しては次のようになります (バージョン1.5以上)。

```yaml
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
```

すべての未認証ユーザに対しては次のようになります (バージョン1.5以上)。

```yaml
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

すべてのユーザに対しては次のようになります (バージョン1.5以上)。

```yaml
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

## デフォルトロールとロール結合 {#default-roles-and-role-bindings}

APIサーバはデフォルトの`ClusterRole`と`ClusterRoleBinding`オブジェクトのセットを作成します。
これらの多くは`system:`接頭辞が付いており、基盤によってそのリソースが"所有"されていることを
示しています。これらのリソースへの変更は機能しないクラスタを招く可能性があります。ひとつの例が
`system:node`クラスタロールです。このロールはkubeletに対する権限を定義しています。このロールが
変更されるとkubeletの動作が阻害される可能性があります。

すべてのデフォルトクラスタロールとロール結合には`kubernetes.io/bootstrapping=rbac-defaults`
というラベルが付与されています。

### 自動調整 {#auto-reconciliation}

起動ごとに、APIサーバはデフォルトクラスタロールに不足の権限と、クラスタロール結合に不足の対象を
更新します。これは不慮の変更からクラスタを回復させ、新しいリリースで変更された権限と対象でロールと
ロール結合を最新に保ちます。

この調整を無効にするには、デフォルトクラスタロールやロール結合の
`rbac.authorization.kubernetes.io/autoupdate`アノテーションを`false`に設定します。デフォルト
権限や対象の不足は機能しないクラスタを招く可能性があることに注意してください。

Kubernetesバージョン1.6以上では、RBAC認可モジュールがアクティブな場合、自動調整が有効になります。

### ロールの検出 {#discovery-roles}

デフォルトロール結合は公にアクセス可能であっても安全と思われるAPI情報の読み取りをすべてのユーザに
対して認可します (CustomResourceDefinitionsも含まれます)。匿名の未認証アクセスを無効にするには、
APIサーバの構成で`anonymous-auth=false`を追加してください。

これらのロールの構成を見るには`kubelet`で以下を実行します。

```
kubectl get clusterroles system:discovery -o yaml
```

メモ: 変更はAPIサーバの再起動時に自動調整を通じて上書きされますのでロールの編集は推奨されません。

<table>
<colgroup><col width="25%"><col width="25%"><col></colgroup>
<tr>
<th>デフォルトClusterRole</th>
<th>デフォルトClusterRoleBinding</th>
<th>説明</th>
</tr>
<tr>
<td><b>system:basic-user</b></td>
<td><b>system:authenticated</b>と<b>system:unauthenticated</b>グループ</td>
<td>ユーザに自身の基本情報への参照のみアクセスを許可します。</td>
</tr>
<tr>
<td><b>system:discovery</b></td>
<td><b>system:authenticated</b>と<b>system:unauthenticated</b>グループ</td>
<td>APIレベルでの検出とネゴシエートに必要なAPI検出エンドポイントへの参照のみアクセスを許可します。</td>
</tr>
</table>

### ユーザ向けロール {#user-facing-roles}

デフォルトロールには`system:`接頭辞のないものもあります。これらはユーザ向けロールを意図しています。
これにはスーパユーザロール (`cluster-admin`) やClusterRoleBindings (`cluster-status`) を使った
クラスタレベルで権限を与えることを意図したロール、RoleBinding (`admin`, `edit`, `view`) を使った
特定の名前空間内で権限を与えることを意図したロールも含まれます。

1.9から、管理者がカスタムリソースのためのルールを含められるようにするため、ユーザ向けロールは
[ClusterRole集約](#aggregated-clusterrole)を使っています。"admin", "edit", "view"ロールにルールを
追加するために、次のラベルを1つ以上持つClusterRoleを作成します。

```yaml
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
```

<table>
<colgroup><col width="25%"><col width="25%"><col></colgroup>
<tr>
<th>デフォルトClusterRole</th>
<th>デフォルトClusterRoleBinding</th>
<th>説明</th>
</tr>
<tr>
<td><b>cluster-admin</b></td>
<td><b>system:masters</b>グループ</td>
<td>すべてのリソースですべてのアクションを実行するために、スーパユーザアクセスを許可します。<b>ClusterRoleBinding</b>で使う場合、クラスタのすべての名前空間にあるすべてのリソースへのフル権限が与えらえます。<b>RoleBinding</b>で使う場合、名前空間自身も含むロール結合の名前空間にあるすべてのリソースへのフル権限が与えられます。</td>
</tr>
<tr>
<td><b>admin</b></td>
<td>None</td>
<td>管理者アクセスを許可します。<b>RoleBinding</b>を使った名前空間内での権限付与を意図しています。<b>RoleBinding</b>で使うと、名前空間内のロールとロール結合の作成を含む、ほとんどのリソースへの読み書きアクセスが許可されます。リソースクオータや名前空間自身への書き込みアクセスは許可されません。</td>
</tr>
<tr>
<td><b>edit</b></td>
<td>None</td>
<td>名前空間内のほとんどのオブジェクトへの読み書きアクセスが許可されます。ロールやロール結合の参照は許可されません。</td>
</tr>
<tr>
<td><b>view</b></td>
<td>None</td>
<td>名前空間内のほとんどのオブジェクトへの参照のみアクセスが許可れます。ロールやロール結合の参照は許可されません。昇格となるためSecretの参照も許可されません。</td>
</tr>
</table>

### コアコンポーネントロール {#core-component-roles}

<table>
<colgroup><col width="25%"><col width="25%"><col></colgroup>
<tr>
<th>デフォルトClusterRole</th>
<th>デフォルトClusterRoleBinding</th>
<th>説明</th>
</tr>
<tr>
<td><b>system:kube-scheduler</b></td>
<td><b>system:kube-scheduler</b>ユーザ</td>
<td>kube-schedulerコンポーネントによって必要なリソースへのアクセスを許可します。</td>
</tr>
<tr>
<td><b>system:volume-scheduler</b></td>
<td><b>system:kube-scheduler</b>ユーザ</td>
<td>kube-schedulerコンポーネントによって必要なボリュームリソースへのアクセスを許可します。</td>
</tr>
<tr>
<td><b>system:kube-controller-manager</b></td>
<td><b>system:kube-controller-manager</b>ユーザ</td>
<td>kube-controller-managerコンポーネントによって必要なリソースへのアクセスを許可します。個々の制御ループで必要な権限は<a href="#controller-roles">コントローラロール</a>に含まれます。</td>
</tr>
<tr>
<td><b>system:node</b></td>
<td>1.8以上はNone</td>
<td><b>すべてのSecretの参照アクセスやすべてのPodのstatusオブジェクトの書き込みアクセスを含む</b>、kubeletコンポーネントによって必要なリソースへのアクセスを許可します。

1.7から、このロールではなく<a href="/ja/docs/reference/access-authn-authz/node/">Node認可モジュール</a>と<a href="/ja/docs/reference/access-authn-authz/admission-controllers/#noderestriction">NodeRestrictionアドミッションプラグイン</a>の使用が推奨され、実行がスケジュールされたPodに基づくkubeletへのAPIアクセスが許可されます。1.7より前は、`system:nodes`グループにこのロールが自動で結合されました。1.7では、`Node`認可モードが有効になっていれば、`system:nodes`グループに結合されました。1.8以降では、結合は作成されません。
</td>
</tr>
<tr>
<td><b>system:node-proxier</b></td>
<td><b>system:kube-proxy</b>ユーザ</td>
<td>kube-proxyコンポーネントによって必要なリソースへのアクセスが許可されます。</td>
</tr>
</table>

### その他のコンポーネントのロール {#other-component-roles}

<table>
<colgroup><col width="25%"><col width="25%"><col></colgroup>
<tr>
<th>デフォルトClusterRole</th>
<th>デフォルトClusterRoleBinding</th>
<th>説明</th>
</tr>
<tr>
<td><b>system:auth-delegator</b></td>
<td>None</td>
<td>移譲された認証と認可チェックが許可されます。これは通常、認証と認可を一元管理するアドオンAPIサーバによって使われます。</td>
</tr>
<tr>
<td><b>system:heapster</b></td>
<td>None</td>
<td><a href="https://github.com/kubernetes/heapster">Heapster</a>コンポーネントのためのロールです。</td>
</tr>
<tr>
<td><b>system:kube-aggregator</b></td>
<td>None</td>
<td><a href="https://github.com/kubernetes/kube-aggregator">kube-aggregator</a>コンポーネントのためのロールです。</td>
</tr>
<tr>
<td><b>system:kube-dns</b></td>
<td><b>kube-system</b>名前空間の<b>kube-dns</b>サービスアカウント</td>
<td><a href="/ja/docs/concepts/services-networking/dns-pod-service/">kube-dns</a>コンポーネントのためのロールです。</td>
</tr>
<tr>
<td><b>system:kubelet-api-admin</b></td>
<td>None</td>
<td>kubelet APIへのフルアクセスが許可されます。</td>
</tr>  
<tr>
<td><b>system:node-bootstrapper</b></td>
<td>None</td>
<td><a href="/ja/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/">Kubelet TLSブートストラップ</a>を実行するのに必要なリソースへのアクセスが許可されます。</td>
</tr>
<tr>
<td><b>system:node-problem-detector</b></td>
<td>None</td>
<td><a href="https://github.com/kubernetes/node-problem-detector">node-problem-detector</a>コンポーネントのためのロールです。</td>
</tr>
<tr>
<td><b>system:persistent-volume-provisioner</b></td>
<td>None</td>
<td><a href="/ja/docs/concepts/storage/persistent-volumes/#provisioner">動的ボリュームプロビジョナ</a>のほとんどが必要とするリソースへのアクセスを許可します。</td>
</tr>
</table>

### コントローラロール {#controller-roles}

[Kubernetesコントローラマネージャ](/ja/docs/admin/kube-controller-manager/)はコア制御ループを実行します。
`--use-service-account-credentials`つきで実行した場合、各制御ループは分離したサービスアカウントを
使って起動します。対応するロールが各制御ループに存在し、`system:controller:`という接頭辞が付きます。
コントローラマネージャが`--use-service-account-credentials`つきで起動されなかった場合、すべての
制御ループは自身の資格情報で実行し、すべての関連するロールが許可されなければなりません。以下のロールが
含まれます。

* system:controller:attachdetach-controller
* system:controller:certificate-controller
* system:controller:cronjob-controller
* system:controller:daemon-set-controller
* system:controller:deployment-controller
* system:controller:disruption-controller
* system:controller:endpoint-controller
* system:controller:generic-garbage-collector
* system:controller:horizontal-pod-autoscaler
* system:controller:job-controller
* system:controller:namespace-controller
* system:controller:node-controller
* system:controller:persistent-volume-binder
* system:controller:pod-garbage-collector
* system:controller:pv-protection-controller
* system:controller:pvc-protection-controller
* system:controller:replicaset-controller
* system:controller:replication-controller
* system:controller:resourcequota-controller
* system:controller:route-controller
* system:controller:service-account-controller
* system:controller:service-controller
* system:controller:statefulset-controller
* system:controller:ttl-controller

## 特権昇格防御とブートストラップ {#privilege-escalation-and-bootstrapping}

RBAC APIはユーザがロールやロール結合を編集することによる特権昇格を防ぎます。これはAPIレベルで強制されるので、
RBAC認証モジュールを使っていなかったとしても適用されます。

ユーザは、以下の事柄の少なくとも1つに該当すれば、ロールを作成/更新できます。

1. 編集されるオブジェクト (`ClusterRole`ならクラスタレベル、`Role`なら同じ名前空間内かクラスタレベル) と
   同じスコープのロールに含まれるすべての権限をすでに持っている
2. `rbac.authorization.k8s.io` APIグループの`roles`や`clusterroles`リソースへ`escalate`メソッドを
   実行する権限を明示的に与えられている (Kubernetes 1.12以降)

例えば、"user-1"がクラスタレベルでSecretを列挙する能力を持っていないのなら、その権限を含む`ClusterRole`を
作ることはできません。ユーザにロールの作成/更新を許可するためには以下のようにします。

1. `Role`または`ClusterRole`オブジェクトの作成/更新を許可するロールを与える
2. そのロールに作成/更新を行う権限を含めるための権限を与える
    * 暗黙的に、これらの権限を与える (許可されていない権限で`Role`や`ClusterRole`を作成または編集しようと
      すると、APIリクエストが拒否される)
    * `rbac.authorization.k8s.io` APIグループの`roles`や`clusterroles`へ`escalate`メソッドを実行する
      権限を与えることで、`Role`や`ClusterRole`に指定した権限を明示的に与える

ロール変更の作成や更新は、参照するロールに含まれるすべての権限を持っているか、参照するロールの`bind`メソッドを
実行するための権限を明示的に与えられている場合のみ行えます。例えば、"user-1"がクラスタレベルでSecretを列挙する
権限を持っていなければ、その権限を与えるロールへの`ClusterRoleBinding`は作成できません。ユーザにロール結合の
作成/更新を許可するためには以下のようにします。

1. `RoleBinding`または`ClusterRoleBinding`オブジェクトの作成/更新を許可するロールを与える
2. 特定のロールへ結合するのに必要な権限を与える
    * 暗黙的には、ロールに含まれる権限を与える
    * 明示的には、特定のロール (またはクラスタロール) の`bind`メソッドを実行する権限を与える

例えば、このクラスタロールとロール結合は"user-1"に他のユーザへ"user-1-namespace"名前空間内で`admin`, `edit`,
`view`ロールを与えることを許可します。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: role-grantor
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["rolebindings"]
  verbs: ["create"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["bind"]
  resourceNames: ["admin","edit","view"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: role-grantor-binding
  namespace: user-1-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: role-grantor
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: user-1
```

最初のロールとロール結合をブートする場合、初期ユーザにまだ持っていない権限を与える必要があります。
初期ロールとロール結合をブートするためには以下のようにします。

* `system:masters`グループの資格情報を使う。これはデフォルト結合によって`cluster-admin`
  スーパユーザロールに結合される
* Insecureポートを有効にしてAPIサーバを実行していれば、そのポートからAPI呼び出しを行うことが
  できます。これは認証や認可を強制しない

## コマンドラインユーティリティ {#command-line-utilities}

名前空間内やクラスタ全体にわたってロールを許可するための`kubectl`コマンドが2つあります。

### `kubectl create rolebinding`

指定した名前空間で`Role`または`ClusterRole`を与えます。以下に例を挙げます。

* "acme"名前空間内の"bob"というユーザに`admin` `ClusterRole`を与える

    ```
    kubectl create rolebindig bob-admin-binding --clusterrole=admin --user=bob --namespace=acme
    ```

* "acme"名前空間内の"myapp"というサービスアカウントに`view` `ClusterRole`を与える

    ```
    kubectl create rolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp --namespace=acme
    ```

### `kubectl create clusterrolebinding`

すべての名前空間を含む、クラスタ全体にわたって`ClusterRole`を与えます。以下に例を挙げます。

* クラスタ全体にわたって、"root"というユーザに`cluster-admin` `ClusterRole`を与える

    ```
    kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root
    ```

* クラスタ全体にわたって、"kubelet"というユーザに`system:node` `ClusterRole`を与える

    ```
    kubectl create clusterrolebinding kubelet-node-binding --clusterrole=system:node --user=kubelet
    ```

* クラスタ全体にわたって、"acme"名前空間の"myapp"というサービスアカウントに`view` `ClusterRole`を与える

    ```
    kubectl create clusterroelbinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp
    ```
    
詳しい使い方はCLIのヘルプを参照してください。

### サービスアカウント権限 {#service-account-permissions}

デフォルトのRBACポリシはコントロールプレーンコンポーネント、ノードコントローラにスコープ付き権限を
許可しますが、`kube-system`名前空間の外にあるサービスアカウントには権限を許可しません (すべての
認証済みユーザに与えられる検出権限を超えます)。

これは必要に応じて、特定のロールを特定のサービスアカウントに与えることを許可します。きめ細かい
ロール結合はセキュリティを向上させますが、管理に労力が必要です。広範囲の許可は不必要な (そして
潜在的に昇格する) APIアクセスをサービスアカウントに与えることになりますが、管理は容易です。

セキュアな度合いが高い順でのアプローチを示します。

1. ロールをアプリケーション固有のサービスアカウントに与える (ベストプラクティス)

    これはPodスペックで`serviceAccountName`を指定する必要があり、(APIやアプリケーションマニフェスト、
    `kubectl create serviceaccount`などで) サービスアカウントを作成する必要があります。
    
    例えば、"my-sa"サービスアカウントに"my-namespace"内の参照のみ権限を与えるには次のようにします。
    
    ```shell
    kubectl create rolebinding my-sa-view \
      --clusterrole=view \
      --serviceaccount=my-namespace:my-sa \
      --namespace=my-namespace
    ```

2. ロールを名前空間内の"default"サービスアカウントに与える

    アプリケーションが`serviceAccountName`を指定しなければ、"default"サービスアカウントを使います。
    
    {{< note >}}**メモ:** "default"サービスアカウントに与えられる権限は、`serviceAccountName`を
    指定しないその名前空間のすべてのPodで利用可能です。{{< /note >}}
    
    例えば、"default"サービスアカウントに"my-namespace"内の参照のみ権限を与えるには次のようにします。
    
    ```shell
    kubectl create rolebinding default-view \
      --clusterrole=view \
      --serviceaccount=my-namespace:default \
      --namespace=my-namespace
    ```

    多くの[アドオン](/ja/docs/concepts/cluster-administration/addons/)は現在`kube-system`名前空間の
    "default"サービスアカウントで実行します。これらのアドオンにスーパユーザアクセスを許可するには、
    `kube-system`名前空間の"default"サービスアカウントにcluster-admin権限を与えます。
    
    {{< note >}}**メモ:** これを有効にすることは、`kube-system`名前空間にAPIへのスーパユーザアクセスを
    許可するSecretが含まれることを意味します。{{< /note >}}
    
    ```shell
    kubectl create clusterrolebinding add-on-cluster-admin \
      --clusterrole=cluster-admin \
      --serviceaccount=kube-system:default
    ```
3. ロールを名前空間のすべてのサービスアカウントに与える

    名前空間のすべてのアプリケーションに、サービスアカウントに関係なくロールを持たせたいのであれば、
    その名前空間のサービスアカウントグループにロールを与えることができます。
    
    例えば、"my-namespace"内のすべてのサービスアカウントにその名前空間の参照のみ権限を与えるには
    次のようにします。
    
    ```shell
    kubectl create rolebinding serviceaccounts-view \
      --clusterrole=view \
      --group=system:serviceaccounts:my-namespace \
      --namespace=my-namespace
    ```

4. 制限されたロールをクラスタ全体のすべてのサービスアカウントに与える (推奨しません)

    名前空間ごとの権限を管理したくないのであれば、クラスタレベルのロールをすべてのサービスアカウントに
    与えることができます。
    
    例えば、クラスタのすべての名前空間のすべてのサービスアカウントに参照のみ権限を与えるには次のように
    します。
    
    ```shell
    kubectl create clusterrolebinding serviceaccounts-view \
      --clusterrole=view \
     --group=system:serviceaccounts
    ```

5. すべてのクラスタレベルのサービスアカウントにスーパユーザアクセスを与える (強く推奨しません)

    権限の分割について全く考慮しないのであれば、すべてのサービスアカウントにスーパユーザアクセスを
    与えることができます。
    
    {{< warning >}}**警告:** これはSecretの参照アクセスやPodを作成する能力のあるユーザにスーパユーザ
    資格情報へのアクセスを許可します。{{< /warning >}}
    
    ```shell
    kubectl create clusterrolebinding serviceaccounts-cluster-admin \
      --clusterrole=cluster-admin \
      --group=system:serviceaccounts
    ```

## 1.5からのアップグレード {#upgrading-from-1-5}

Kubernetes 1.6より前では、多くのDeploymentが、すべてのサービスアカウントへのフルAPIアクセスの許可を
含む、非常に寛大なABACポリシーを使っていました。

デフォルトのRBACポリシーはコントロールプレーンコンポーネント、ノードコントローラにスコープ付き権限を
許可しますが、`kube-system`名前空間の外にあるサービスアカウントには権限を許可しません (すべての
認証済みユーザに与えられる検出権限を超えます)。

はるかにセキュアですが、これは自動でAPI権限を受け取ることを期待している既存のワークロードに破壊的な
影響を与えます。この移行を管理するためのアプローチが2つあります。

### 並列認可モジュール {#parallel-authorizers}

RBACとABAC認可モジュールを両方とも実行し、
[レガシーABACポリシー](/ja/docs/reference/access-authn-authz/abac/#policy-file-format)を含む
ポリシーファイルを指定します。

```
--authorization-mode=RBAC,ABAC --authorization-policy-file=mypolicy.json
```

RBAC認可モジュールが最初にリクエストを認可しようとします。APIリクエストが拒否されれば、次にABAC
認可モジュールが実行されます。これはRBACかABACの *どちらか* で許可されたリクエストが許可されることを
意味します。

ログレベル2以上 (`--v=2`) で実行する場合、RBAC拒否のログをapiserverのログで見ることができます
(接頭辞`RBAC DENY:`が付いています)。この情報はどのロールをどのユーザ、グループ、サービスアカウントに
与える必要があるのかを決めるために使えます。
[ロールをサービスアカウントに与える](#service-account-permissions)と、ワークロードはRBAC拒否メッセージ
なしで実行するようになり、ABAC認証モジュールを削除できます。

## 寛大なRBAC権限 {#permissive-rbac-permissions}

RBACロール結合を使って寛大なポリシーを複製できます。

{{< warning >}}
**警告:** 以下のポリシーは **すべての** サービスアカウントにクラスタ管理者としてのふるまいを許可します。
コンテナで実行しているあらゆるアプリケーションは自動でサービスアカウントの資格情報を受け取り、Secretの
閲覧や権限の編集を含むAPIに対するあらゆるアクションを実行できます。これは推奨されるポリシーではありません。

```
kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts
```
{{< /warning >}}

{{% /capture %}}
