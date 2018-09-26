---
title: 認可の概要
content_template: templates/concept
weight: 60
---

{{% capture overview %}}
サポートされた認可モジュールを使ったポリシー作成の詳細を含む、Kubernetesの認可について学んでください。
{{% /capture %}}

{{% capture body %}}
Kubernetesでは、リクエストを認可する (アクセスを許可する) 前に認証 (ログイン) しなければなりません。認証についての詳細は[アクセス制御の概要](/docs/reference/access-authn-authz/controlling-access/)を参照してください。

KubernetesはREST APIリクエストで一般的な属性が来ることを期待します。これは、Kubernetesの認可が、Kubernetes以外のAPIを扱う既存の組織レベルもしくはクラウドプロバイダレベルのアクセス制御システムと連携することを意味します。

## リクエストの許可/拒否の決定 {#determine-whether-a-request-is-allowed-or-denied}

KubernetesはAPIサーバを使ってリクエストを認可します。APIサーバはすべてのポリシーに対して、すべてのリクエスト属性を評価し、リクエストを許可または拒否します。処理を進めるにはAPIリクエストのすべての部分が許可されなければなりません。これは許可はデフォルトで拒否されることを意味します。

(KubernetesはAPIサーバを使いますが、特定のオブジェクトの特定のフィールドのアクセス制御とポリシーはアドミッションコントローラによって扱われます。)

複数の認可モジュールが構成される場合、それぞれは順番にチェックされます。いずれかの認可モジュールがリクエストを認可まらは拒否すれば、その決定は即座に返され、他の認可モジュールは調べられません。もしすべてのモジュールがリクエストに対する判定を行わなければ、そのリクエストは拒否されます。拒否はHTTPステータスコード403を返します。

## リクエスト属性の検査 {#review-your-request-attributes}

Kubernetesは以下のAPIリクエスト属性のみを検査します。

 * **user** - 認証中に提供される`user`文字列
 * **group** - 認証されたユーザが所属するグループ名のリスト
 * **extra** - 認証レイヤーで提供される任意の文字列キーと文字列値のマップ
 * **API** - リクエストがAPIリソースに対するものがどうかを指示する
 * **Request path** - `/api`や`/healthz`のような雑多な非リソースエンドポイントへのパス
 * **API request verb** - API verb `get`, `list`, `create`, `update`, `patch`, `watch`, `proxy`, `redirect`, `delete`, `deletecollection`がリソース要求で使われます。リソースAPIエンドポイントに対するAPI Verbの決定は以下の[リクエストVerbの決定](/ja/docs/reference/access-authn-authz/authorization/#determine-the-request-verb)を参照してください
 * **Reource** - アクセスされるリソースの名前またはID (リソース要求のみ) -- `get`, `update`, `patch`, `delete` を使うリソース要求に対して、リソース名を指定しなければなりません。
 * **Subresource** - アクセスされるサブリソース (リソース要求のみ)
 * **Namespace** - アクセスされるオブジェクトの名前空間 (名前空間のあるリソース要求のみ)
 * **API group** - アクセスされるAPIグループ (リソース要求のみ)。空の文字列は[core APIグループ](/ja/docs/concepts/overview/kubernetes-api/)を指定します。

## リクエストメソッドの決定 {#determine-the-request-verb}

リソースAPIエンドポイントに対するリクエストメソッドを決定するために、使われるHTTPメソッドと、リクエストが個々のリソースかリソースの集合に作用するかどうかを検査します。

HTTPメソッド | リクエストメソッド
----------|---------------
POST      | create
GET, HEAD | get (個々のリソース), list (集合)
PUT       | update
PATCH     | patch
DELETE    | delete (個々のリソース), deletecollection (集合)

Kubernetesは特殊なメソッドを使う権限に対する認可をチェックすることがあります。以下の例があります。

* [PodSecurityPolicy](/ja/docs/concepts/policy/pod-security-policy/)は`policy` APIグループの`podsecuritypolicies`で`use`メソッドの認可をチェックします
* [RBAC](/ja/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping)は`rbac.authorization.k8s.io` APIグループの`roles`と`clusterroles`で`bind`メソッドの認可をチェックします
* [認証](/ja/docs/reference/access-authn-authz/authentication/)レイヤーはコアAPIグループの`users`、`group`、`serviceaccounts`と`authentication.k8s.io` APIグループの`userextras`で`impersonate`メソッドの認可をチェックします

## 認可モジュール {#authorization-modules}

 * **Node** - 実行がスケジュールされるPodに基づいてkubeletの権限を許可する特殊目的の認可モジュールです。Node認可モードについての詳細は、[Node認可](/docs/reference/access-authn-authz/node/)を参照してください
 * **ABAC** - 属性ベースアクセス制御 (Attribute-Based Access Control; ABAC) は、属性の組み合わせポリシーでアクセス権がユーザに与えられるアクセス制御の枠組みです。ポリシーにはあらゆるタイプの属性 (ユーザ属性、リソース属性、オブジェクト、環境属性など) が使えます。ABACモードについての詳細は[ABACモード](/ja/docs/reference/access-authn-authz/abac/)を参照してください
 * **RBAC** - ロールベースアクセス制御 (Role-Based Access Control; RBAC) は、企業での個々のユーザの役割に基づいて計算資源やネットワークリソースへのアクセスを制御する方法です。この背景には、アクセスは特定のタスク (ファイルの閲覧や作成、編集など) を実行する個々のユーザの能力だという考え方があります。RBACモードについての詳細は[RBACモード](/ja/docs/reference/access-authn-authz/rbac/)を参照してください
   * 特定のRBACが認可の決定を行うために`rbac.authorization.k8s.io` APIグループを使う場合、管理者にKubernetes APIを通じた権限ポリシーの動的な構成を許可します
   * RBACを有効にするためには、`--authorization-mode=RBAC`を指定してAPIサーバを起動します
 * **Webhook** - WebhookはHTTPのコールバックで、何かが起きた時にHTTP POSTが発生するという、シンプルなイベント通知です。Webhookを実装するWebアプリケーションは特定の事柄が発生した時にURLへメッセージをPOSTします。Webhookモードについての詳細は[Webhookモード](/ja/docs/reference/access-authn-authz/webhook/)を参照してください

#### APIアクセスのチェック {#checking-api-access}

`kubectl`はAPI認可レイヤーに素早く問い合わせを行うための`auth can-i`サブコマンドを提供しています。このコマンドは現在のユーザが与えられたアクションを実行できるかどうかを判定するために`SelfSubjectAccessReview` APIを使い、使われている認可モードにかかわらず動作します。

```bash
$ kubectl auth can-i create deployments --namespace dev
yes
$ kubectl auth can-i create deployments --namespace prod
no
```

管理者は[ユーザ切り替え](/docs/reference/access-authn-authz/authentication/#user-impersonation)と組み合わせて、他のユーザが実行できるアクションを判定できます。

```bash
$ kubectl auth can-i list secrets --namespace dev --as dave
no
```

`SelfSubjectAccessReview`は`authorization.k8s.io` APIグループの一部で、外部サービスにAPIサーバ認可を公開します。このグループに含まれる他のリソースには次のようなものがあります。

* `SubjectAccessReview` - 現在のユーザだけでなく、あらゆるユーザに対するアクセス検査です。認可の判定をAPIサーバに移譲する際に有用です。例えばkubeletと拡張APIサーバは自身のAPIへのユーザアクセスを判定するためにこれを使います
* `LocalSubjectAccessReview` - `SubjectAccessReview`と似ていますが、特定の名前空間に制限されます
* `SelfSubjectRulesReview` - 名前空間内でユーザが実行可能なアクションの一覧を返す検査です。ユーザが自身のアクセス権を概観したり、UIに対してアクションの表示/非表示を切り替えたりするのに有用です

これらのAPIは通常のKubernetesリソースを作成することで問い合わせることができます。レスポンスの"status"フィールドが問い合わせ結果になります。

```bash
$ kubectl create -f - -o yaml << EOF
apiVersion: authorization.k8s.io/v1
kind: SelfSubjectAccessReview
spec:
  resourceAttributes:
    group: apps
    resource: deployments
    verb: create
    namespace: dev
EOF

apiVersion: authorization.k8s.io/v1
kind: SelfSubjectAccessReview
metadata:
  creationTimestamp: null
spec:
  resourceAttributes:
    group: apps
    resource: deployments
    namespace: dev
    verb: create
status:
  allowed: true
  denied: false
```

## 認可モジュールに対してフラグを使う {#using-flags-for-your-authorization-module}

どの認可モジュールを使うのか指示するために、ポリシーにフラグを含めなければなりません。

以下のフラグが使えます。

  * `--authorization-module-mode=ABAC` 属性ベースアクセス制御 (ABAC) はローカルファイルを使ってポリシーを構成することを許可します
  * `--authorization-module-mode=RBAC` ロールベースアクセス制御 (RBAC) はKubernetes APIを使ってポリシーの作成と格納することを許可します
  * `--authorization-module-mode=Webhook` WebHookはリモートRESTエンドポイントを使って認可を管理することを許可するHTTPコールバックモードです
  * `--authorization-module-mode=Node` Node認可はkubeletによて作成されたAPIリクエストを認可する特殊用途の認可モードです
  * `--authorization-module-mode=AlwaysDeny` すべてのリクエストをブロックします。このフラグはテスト目的でのみ使います
  * `--authorization-module-mode=AlwaysAllow` すべてのリクエストを許可します。APIリクエストを認可する必要がない場合にのみ使います

複数の認可モジュールを選択することもできます。モジュールは順番にチェックされるので。先に指定らえたモジュールの優先度が高くなります。

## Podの作成を通じた権限昇格 {#privilege-escalation-via-pod-creation}

名前空間内にPodを作成できる能力を持つユーザは、その名前空間内で潜在的に特権昇格することができます。そのユーザはその名前空間で特権にアクセスできるPodを作ることができます。そのユーザが参照できないSecretにアクセスしたり、異なる権限をもつサービスアカウントで実行したりするPodを作成できます。

{{< caution >}}
**注意:** システム管理者は注意してPodの作成権を与えてください。ある名前空間にPod (もしくはPodを作成するコントローラ) を作成する権限を持つユーザは、その名前空間のすべてのSecretやConfigMapを参照でき、あらゆるサービスアカウントに切り替えることができ、そのアカウントが取ることのできるあらゆるアクションを取ることができます。これは認可モードにかかわらず適用されます。
{{< /caution >}}
{{% /capture %}}

{{% capture whatsnext %}}
* 認証についての詳細は、[Kubernetes APIへのアクセス制御](/ja/docs/reference/access-authn-authz/controlling-access/)の **認証** を参照してください
* アドミッションコントロールについての詳細は、[アドミッションコントローラの使用](/ja/docs/reference/access-authn-authz/admission-controllers/)を参照してください
{{% /capture %}}
