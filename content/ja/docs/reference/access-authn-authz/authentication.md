---
title: 認証
content_template: templates/concept
weight: 10
---

{{% capture overview %}}
このページでは認証の概要を提供します。
{{% /capture %}}

{{% capture body %}}
## Kubernetesのユーザ

すべてのKubernetesクラスタは2カテゴリのユーザを持ちます。Kubernetesによって管理されるサービスアカウントと通常のユーザです。

通常のユーザは外部の、独立したサービスによって管理されると思われます。秘密鍵を配布する管理者、KeystoneやGoogle Accountsといったユーザストア、ユーザ名とパスワードのリスト。つまり _Kubernetesは通常のユーザアカウントを表現するオブジェクトを持ちません。_ 通常ユーザはAPI呼び出しを通じてクラスタに追加することはできません。

対照的に、サービスアカウントはKubernetes APIによって管理されるユーザです。これらは特定の名前空間に結合され、APIサーバによって自動的に、もしくはAPI呼び出しを通じて手動で作成されます。サービスアカウントは`Secrets`として格納された資格情報の集合に結びつけられ、これはKubernetes APIとやりとりするクラスタ内のプロセスを許可するPodにマウントされます。

APIリクエストは通常ユーザかサービスアカウントに結びつけられ、そうしなければ匿名リクエストとして扱われます。これはワークステーションで`kubectl`をタイプする人間のユーザから、ノードの`kubelet`、コントロールプレーンのメンバーまで、クラスタの内部または外部のすべてのプロセスは、APIサーバへのリクエストを作成するときに認証しなければならず、そうでなければ匿名ユーザとして扱われることを意味します。

## 認証戦略

Kubernetesは認証プラグインを通じてAPIリクエストを認証するためにクライアント証明書や、無記名トークン、認証プロキシ、もしくはHTTPベーシック認証を使います。HTTPリクエストがAPIサーバへ到達すると、プラグインは以下の属性をリクエストに関連付けようとします。

* ユーザ名: エンドユーザを識別する文字列。一般的な値は`kube-admin`や`jane@example.com`といったものです。
* UID: エンドユーザを識別し、ユーザ名よりも一貫性と一意性のある文字列。
* グループ: 通常、グループ化されたユーザの集合をユーザと関連付ける文字列の集合。
* 追加フィールド: 認可モジュールに便利だと思われる追加情報を含む文字列のリストへの文字列のマップ。

すべての値は認証システムには不透明で、[認可モジュール](/ja/docs/reference/access-authn-authz/authorization/)によって解釈された重要性のみを保持します。

一度に複数の認証方法を有効にできます。通常は少なくとも2つの方法を使うべきです。

 - サービスアカウントに対するサービスアカウントトークン
 - ユーザ認証のための少なくとも1つ別の方法

複数の認証モジュールが有効になっていると、正常にリクエストを認証する最初のモジュールは評価を省きます。APIサーバは認証モジュールが実行される順番を保証しません。

`system:authenticated`グループがすべての認証されたユーザのグループのリストに含まれます。

他の認証プロトコル (LDAP, SAML, Kerberos, 代替x509スキームなど) との統合は [認証プロキシ](#認証プロキシ)や[認証Webhook](#webhookトークン認証)を使って遂行されます。

### X509クライアント証明書

クライアント証明書認証はAPIサーバに`--client-ca-file=SOMEFILE`を渡すことで有効にできます。参照されるファイルは、APIサーバに提示されるクライアント証明書を認証するのに使われる1つ以上の証明機関を含まなければなりません。クライアント証明書が提示され認証されると、サブジェクトのコモンネームがリクエストに対するユーザ名として使われます。Kubernetes 1.4から、クライアント証明書は証明書の組織フィールドを使ってユーザのグループを指定できるようにもなりました。ユーザを複数グループに含めるためには、証明書に複数の組織フィールドを含めます。

例えば、証明書署名要求を作成するために`openssl`コマンドラインツールを使います。

``` bash
openssl req -new -key jbeda.pem -out jbeda-csr.pem -subj "/CN=jbeda/O=app1/O=app2"
```

これは2つのグループ"app1", "app2"に所属する、ユーザ名"jbeda"のCSRを作成します。

クライアント証明書の生成方法については、[証明書の管理](/ja/docs/concepts/cluster-administration/certificates/)を参照してください。

### 静的トークンファイル

APIサーバはコマンドラインで`--token-auth-file=SOMEFILE`が与えられるとファイルから無記名トークンを読み込みます。現在、トークンは無期限に有効で、トークンリストはAPIサーバの再起動なしに変更することはできません。

トークンファイルは最小3列のcsvファイルです。トークン、ユーザ名、ユーザUID、そしてオプションのグループ名が続きます。

{{< note >}}
**メモ:** 複数のグループがあるのなら、二重引用符で囲まなければなりません。

```conf
token,user,uid,"group1,group2,group3"
```
{{< /note >}}

#### 無記名トークンをリクエストに追加する

HTTPクライアントから無記名トークン認証を使うとき、APIサーバは`Bearer THETOKEN`の値を持つ`Authrization`ヘッダがあることを期待します。無記名トークンは、HTTPのエンコーディングやクオーティングの仕組みを使わないHTTPヘッダ値を設定できる文字列でなければなりません。たとえば、無記名トークンが`31ada4fd-adec-460c-809a-9e56ceb75269`であれば、HTTPヘッダは以下のようになります。

```http
Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269
```

### ブートストラップトークン

この機能は現在 **アルファ** です。

新しいクラスタに対して最新のブートストラッピングを許可するために、Kubernetesは *ブートストラップトークン* という動的に管理される無記名トークン型を含みます。これらのトークンは`kube-system`名前空間のSecretとして格納され、動的に管理、作成することができます。コントローラマネージャには無効になったブートストラップトークンを削除するTokenCleanerコントローラが含まれます。

トークンは`[a-z0-9]{6}.[a-z0-9]{16}`の形式です。最初の部分はトークンIDで2つ目の部分はトークンシークレットです。HTTPヘッダでは次のようにトークンを指定します。

```http
Authorization: Bearer 781292.db7bc3a58fc5f07e
```

APIサーバで`--experimental-bootstrap-token-auth`を使ってブートストラップトークン認証モジュールを有効にしなければなりません。コントローラマネージャの`--controllers`フラグ経由でTokenCleanerコントローラを有効にしなければなりません。これは`--controllers=*,tokencleaner`のようにします。クラスタを立ち上げるために`kubeadm`を使ったのであれば、`kubeadm`がこれを行います。

認証モジュールは`system:bootstrap:<Token ID>`として認証します。これは`system:bootstrap`グループに含まれます。ユーザにブートストラッピングを過ぎてこれらのトークンを利用させないようにするため、名前とグループは意図的に制限されています。ユーザ名とグループはクラスタを立ち上げるサポートを行うための適切な認可ポリシを作るために (`kubeadm`によって) 使われます。

`kubeadm`でこれらのトークンを管理する方法と併せてブートストラップトークン認証モジュールとコントローラについての詳細なドキュメントについては、[ブートストラップトークン](/docs/reference/access-authn-authz/bootstrap-tokens/)を参照してください。

### 静的パスワードファイル

ベーシックん印象は `--basic-auth-file=SOMEFILE`オプションをAPIサーバに渡すことで有効になります。現在ベーシック認証資格情報は無制限に有効で、APIサーバの再起動なしにパスワードを変更することはできません。ベーシック認証は現在、利便性のためにサポートされていますが、上で述べたようなよりセキュアなモードをより使いやすくしました。

ベーシック認証のファイルは最小で3列 (パスワード、ユーザ名、ユーザID) のCSVファイルです。Kubernetes 1.6以降、カンマ区切りのグループ名を含む4行目を指定できます。複数のグループを指定する場合、4列目の値は二重引用符で囲まなければなりません。下の例を参照してください。

```conf
password,user,uid,"group1,group2,group3"
```

HTTPクライアントからベーシック認証を使う時、APIサーバは`Basic BASE64ENCODED(USER:PASSWORD)`の値を持つ`Authorization`ヘッダがあることを期待します。

### サービスアカウントトークン

サービスアカウントはリクエストを認証するために署名された無記名トークンを使う自動的に有効になる認証モジュールです。このプラグインは2つのオプションのフラグを取ります。

* `--service-account-key-file` 無記名トークンを署名するPEMエンコードされたキーを含むファイルです。指定されなければ、APIサーバのTLS秘密鍵が使われます。
* `--service-account-lookup` 有効であれば、APIから削除されたトークンは失効されます。

サービスアカウントは通常APIサーバによって自動的に作られ、`ServiceAccount`[アドミッションコントローラ](/ja/docs/reference/access-authn-authz/admission-controllers/)を通じてクラスタで実行するPodに関連付けられます。無記名トークンはPodの周知の位置にマウントされ、クラスタ内のプロセスがAPIサーバとやりとりできるようになります。アカウントは`PodSpec`の`serviceAccountName`フィールドを使ってPodと明示的に関連付けさせることもできます。

{{< note >}}
**メモ:** `serviceAccountName`は自動的に行われるため通常は省略されます。
{{< /note >}}

```yaml
apiVersion: apps/v1 # このapiVersionはKubernetes 1.9以降
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 3
  template:
    metadata:
    # ...
    spec:
      serviceAccountName: bob-the-bot
      containers:
      - name: nginx
        image: nginx:1.7.9
```

サービスアカウント無記名トークンはクラスタの外部で使っても完全に有効で、Kubernetes APIを使いたい長時間のJobに対するIDを作るために使うことができます。サービスアカウントを手動で作るためには、単純に`kubectl create serviceaccount (NAME)`コマンドを使います。これは現在の名前空間にサービスアカウントと関連するSecretを作ります。

```
$ kubectl create serviceaccount jenkins
serviceaccount "jenkins" created
$ kubectl get serviceaccounts jenkins -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  # ...
secrets:
- name: jenkins-token-1yvwg
```

作成されたSecretはAPIサーバの公開CAと署名されたJSON Web Token (JWT)を保持します。

```
$ kubectl get secret jenkins-token-1yvwg -o yaml
apiVersion: v1
data:
  ca.crt: (APISERVER'S CA BASE64 ENCODED)
  namespace: ZGVmYXVsdA==
  token: (BEARER TOKEN BASE64 ENCODED)
kind: Secret
metadata:
  # ...
type: kubernetes.io/service-account-token
```

{{< note >}}
**メモ:** Secretは常にbase64エンコードされるので、値はbase64でエンコードされています。
{{< /note >}}

署名されたJWTは与えらえたサービスアカウントとして認証するための無記名トークンとして使われます。トークンをリクエストに含める方法は[上](#無記名トークンをリクエストに追加する)を参照してください。通常、これらのSecretはAPIサーバへのクラスタ内アクセスのためにPodにマウントされますが、クラスタ外からも同様に使えます。

サービスアカウントはユーザ名 `system:serviceaccount:(NAMESPACE):(SERVICEACCOUNT)`で認証し、グループ `system:serviceaccounts`と`system:serviceaccounts:(NAMESPACE)`に割り当てます。

警告: サービスアカウントトークンはSecretに格納されるので、これらのSecretへの参照権限を持つユーザはサービスアカウントとして認証できます。サービスアカウントの権限をSecretへの参照権限を与える時は注意してください。

### OpenID Connectトークン

[OpenID Connect](https://openid.net/connect/)は特にAzure Active DirectoryやSalesforce, GoogleといったいくつかのOAuth2プロバイダがサポートするOAuth2の拡張です。このプロトコルのOAuth2からの主な拡張は、[IDトークン](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)と呼ばれるアクセストークンを返す追加フィールドです。このトークンは、ユーザのメールアドレスといったよく知られたフィールドを持つサーバによって署名された JSON Web Token (JWT) です。

ユーザを識別するために、認証モジュールはOAuth2[トークンレスポンス](https://openid.net/specs/openid-connect-core-1_0.html#TokenResponse)からの`id_token` (`access_token`ではない) を無記名トークンとして使います。トークンをリクエストに含める方法は[上](#無記名トークンをリクエストに追加する)を参照してください。

![Kubernetes OpenID Connect Flow](/images/docs/admin/k8s_oidc_login.svg)

1. IDプロバイダにログインする
2. IDプロバイダは`access_token`, `id_token`, `refresh_token`を提供する
3. `kubectl`を使うのであれば、`id_token`を`--token`フラグに使うか、直接`kubeconfig`に追加する
4. `kubectl`はAPIサーバにAuthoriztionというヘッダに`id_token`をつけて送信する
5. APIサーバは構成に書かれた証明書をチェックすることで JWT署名を確認する
6. `id_token`が期限切れでないことを確認する
7. ユーザが認可されることを確認する
8. 一度APIサーバに認可されると `kubectl`へレスポンスを返す
9. `kubectl`はユーザにフィードバックをする

ユーザを認証するために必要なすべてのデータは`id_token`にあるので、KubernetesはIDプロバイダに連絡する必要はありません。すべてのリクエストがステートレスなこのモデルでは、認証に対して非常にスケーラブルなソリューションを提供します。いくつかの課題を提示します。

1. Kubernetesは認証プロセスを開始するための"Webインタフェース"を持ちません。資格情報を収集するブラウザやインタフェースがないので、最初にIDプロバイダへ認証する必要があります。
2. `id_token`は失効させられず、証明書のようなものであるので、有効期限を短く (数分程度) しなければならないので、数分ごとに新しいトークンを得る必要がある点が非常に悩ましくなる可能性があります。
3. `kubectl proxy`コマンドまたは`id_token`を注入するリバースプロキシを使わずにKubernetesダッシュボードへの認証を行う容易な方法はありません。

#### APIサーバの構成

プラグインを有効にするために、APIサーバで以下のフラグを構成します。

| Parameter | Description | Example | Required |
| --------- | ----------- | ------- | ------- |
| `--oidc-issuer-url` | APIサーバが公開鍵を探すことのできるプロバイダのURL。`https://`スキームを使うURLのみが許される。通常は、"https://accounts.google.com" や "https://login.salesforce.com" といったパスなしのプロバイダの探索URLである。このURLは.wel-known/openid-configrationの下のレベルを指すべきである。 | 探索URLが `https://accounts.google.com/.well-known/openid-configuration` であれば, 値は `https://accounts.google.com` である。 | Yes |
| `--oidc-client-id` |  すべてのトークンが発行されるクライアントID | kubernetes | Yes |
| `--oidc-username-claim` | ユーザ名として使うためのJWT要求。デフォルトでは`sub`で、エンドユーザの一意な識別子であることが期待される。管理者はプロバイダに応じて`email`や`name`のような他の要求を選択することができる。しかし、`email`以外の要求は他のプラグインと名前の衝突を避けるために発行者のURLを接頭辞としてつけるようにする。 | sub | No |
| `--oidc-username-prefix` | (`system:`ユーザのような) 既存の名前との衝突を避けるためのユーザ名要求につけられる接頭辞。例えば、`oidc:`という値は`oidc:jane.doe`といったユーザ名を作成する。このフラグが指定されず、`--oidc-user-claim`が`email`でない場合、`( Issuer URL)#`がデフォルトの接頭辞となる `( Issuer URL )`は`--oidc-issuer-url`の値)。`-`という値はすべての接頭辞を無効にするために使われる。. | `oidc:` | No |
| `--oidc-groups-claim` | ユーザのグループとして使われる JWT要求。この要求が存在すれば、文字列の配列でなければならない。 | groups | No |
| `--oidc-groups-prefix` | (`system:`グループのような) 既存の名前との衝突を避けるための、グループにつけられる接頭辞。例えば、値 `oicd:`であれば、`oidc:engineering`や`oidc:infra`といったグループ名が作成される。 | `oidc:` | No |
| `--oidc-ca-file` | IDプロバイダのWeb証明書を署名しCAに対する証明書へのパス。デフォルトはホストのルートCA。 | `/etc/kubernetes/ssl/kc-ca.pem` | No |

APIサーバはOAuth2クライアントではなく、単一の発行者を信用するようにしか構成できません。これにより、サードパーティによって発行された資格情報を信じることなく、Googleのような公開プロバイダを使うことができます。複数のOAuthクライアントを使いたい管理者は、他のプロバイダのかわりに1つのクライアントがトークンを発行できる仕組みである`azp` (認可パーティ) をサポートするプロバイダを探すべきです。

KubernetesはOpenID Connect IDプロバイダを提供しません。既存のOpenID Connect IDプロバイダ (Googleや[その他](http://connect2id.com/products/nimbus-oauth-openid-connect-sdk/openid-connect-providers)のような) 使用してください。もしくは、CoreOSの[dex](https://github.com/coreos/dex)や, [Keycloak](https://github.com/keycloak/keycloak)、CloudFoundryの [UAA](https://github.com/cloudfoundry/uaa), or Tremolo Security の [OpenUnison](https://github.com/tremolosecurity/openunison)などのような自身のIDプロバイダを実行することができます。

Kubernetesで利用するIDプロバイダは以下の条件を満たす必要があります。

1. [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html)をサポートする
2. 古い暗号を使わないTLSで実行する
3. CAに署名された証明書を持つ (CAは商用CAでなくても、自己署名でもよい)

上の必要条件3で、CAに署名された証明書が必要なことに注意してください。(GoogleやMicrosoftのようなクラウドプロバイダではなく) 自身のIDプロバイダをデプロイするのであれば、自己署名でもよいので、`CA`フラグを`TRUE`に設定した証明書によって署名されたIDプロバイダのWebサーバ証明書を持たなければなりません。使いやすいCAがなければ、CoreOSチームが作った、簡単なCAと署名された証明書と鍵のペアを生成する[このスクリプト](https://github.com/coreos/dex/blob/1ee5920c54f5926d6468d2607c728b71cfe98092/examples/k8s/gencert.sh)が使えます。または、より長い有効期限で鍵長の長いSHA256証明書を生成する[このスクリプト](https://raw.githubusercontent.com/TremoloSecurity/openunison-qs-kubernetes/master/src/main/bash/makessl.sh)が使えます。

特定のシステムに対するセットアップ説明は以下を参照してください。

- [UAA](http://apigee.com/about/blog/engineering/kubernetes-authentication-enterprise)
- [Dex](https://speakerdeck.com/ericchiang/kubernetes-access-control-with-dex)
- [OpenUnison](https://github.com/TremoloSecurity/openunison-qs-kubernetes)

#### kubectlを使う

##### Option 1 - OIDC認証モジュール

最初のオプションはkubectlの`oidc`認証モジュールを使うことで、これはすべてのリクエストに無記名トークンとして`id_token`を設定し、トークンの期限が切れると更新します。プロバイダにログインした後、`id_token`、`refresh_token`、`client_id`、`client_secret`を追加し、プラグインを構成するためにkubectlを使います。

リフレッシュトークンレスポンスの一部として`id_token`を返さないプロバイダ (例えば[Okta](https://developer.okta.com/docs/api/resources/oidc.html#response-parameters-4)) はこのプラグインではサポートされず、次の"Option 2"を使うべきです。

```bash
kubectl config set-credentials USER_NAME \
   --auth-provider=oidc \
   --auth-provider-arg=idp-issuer-url=( issuer url ) \
   --auth-provider-arg=client-id=( your client id ) \
   --auth-provider-arg=client-secret=( your client secret ) \
   --auth-provider-arg=refresh-token=( your refresh token ) \
   --auth-provider-arg=idp-certificate-authority=( path to your ca certificate ) \
   --auth-provider-arg=id-token=( your id_token )
```

例のように、IDプロバイダへ認証した後、以下のコマンドを実行します。

```bash
kubectl config set-credentials mmosley  \
        --auth-provider=oidc  \
        --auth-provider-arg=idp-issuer-url=https://oidcidp.tremolo.lan:8443/auth/idp/OidcIdP  \
        --auth-provider-arg=client-id=kubernetes  \
        --auth-provider-arg=client-secret=1db158f6-177d-4d9c-8a8b-d36869918ec5  \
        --auth-provider-arg=refresh-token=q1bKLFOyUiosTfawzA93TzZIDzH2TNa2SMm0zEiPKTUwME6BkEo6Sql5yUWVBSWpKUGphaWpxSVAfekBOZbBhaEW+VlFUeVRGcluyVF5JT4+haZmPsluFoFu5XkpXk5BXqHega4GAXlF+ma+vmYpFcHe5eZR+slBFpZKtQA= \
        --auth-provider-arg=idp-certificate-authority=/root/ca.pem \
        --auth-provider-arg=id-token=eyJraWQiOiJDTj1vaWRjaWRwLnRyZW1vbG8ubGFuLCBPVT1EZW1vLCBPPVRybWVvbG8gU2VjdXJpdHksIEw9QXJsaW5ndG9uLCBTVD1WaXJnaW5pYSwgQz1VUy1DTj1rdWJlLWNhLTEyMDIxNDc5MjEwMzYwNzMyMTUyIiwiYWxnIjoiUlMyNTYifQ.eyJpc3MiOiJodHRwczovL29pZGNpZHAudHJlbW9sby5sYW46ODQ0My9hdXRoL2lkcC9PaWRjSWRQIiwiYXVkIjoia3ViZXJuZXRlcyIsImV4cCI6MTQ4MzU0OTUxMSwianRpIjoiMm96US15TXdFcHV4WDlHZUhQdy1hZyIsImlhdCI6MTQ4MzU0OTQ1MSwibmJmIjoxNDgzNTQ5MzMxLCJzdWIiOiI0YWViMzdiYS1iNjQ1LTQ4ZmQtYWIzMC0xYTAxZWU0MWUyMTgifQ.w6p4J_6qQ1HzTG9nrEOrubxIMb9K5hzcMPxc9IxPx2K4xO9l-oFiUw93daH3m5pluP6K7eOE6txBuRVfEcpJSwlelsOsW8gb8VJcnzMS9EnZpeA0tW_p-mnkFc3VcfyXuhe5R3G7aa5d8uHv70yJ9Y3-UhjiN9EhpMdfPAoEB9fYKKkJRzF7utTTIPGrSaSU6d2pcpfYKaxIwePzEkT4DfcQthoZdy9ucNvvLoi1DIC-UocFD8HLs8LYKEqSxQvOcvnThbObJ9af71EwmuE21fO5KzMW20KtAeget1gnldOosPtz1G5EwvaQ401-RPQzPGMVBld0_zMCAwZttJ4knw
```

これは以下のような構成を作成します。

```yaml
users:
- name: mmosley
  user:
    auth-provider:
      config:
        client-id: kubernetes
        client-secret: 1db158f6-177d-4d9c-8a8b-d36869918ec5
        id-token: eyJraWQiOiJDTj1vaWRjaWRwLnRyZW1vbG8ubGFuLCBPVT1EZW1vLCBPPVRybWVvbG8gU2VjdXJpdHksIEw9QXJsaW5ndG9uLCBTVD1WaXJnaW5pYSwgQz1VUy1DTj1rdWJlLWNhLTEyMDIxNDc5MjEwMzYwNzMyMTUyIiwiYWxnIjoiUlMyNTYifQ.eyJpc3MiOiJodHRwczovL29pZGNpZHAudHJlbW9sby5sYW46ODQ0My9hdXRoL2lkcC9PaWRjSWRQIiwiYXVkIjoia3ViZXJuZXRlcyIsImV4cCI6MTQ4MzU0OTUxMSwianRpIjoiMm96US15TXdFcHV4WDlHZUhQdy1hZyIsImlhdCI6MTQ4MzU0OTQ1MSwibmJmIjoxNDgzNTQ5MzMxLCJzdWIiOiI0YWViMzdiYS1iNjQ1LTQ4ZmQtYWIzMC0xYTAxZWU0MWUyMTgifQ.w6p4J_6qQ1HzTG9nrEOrubxIMb9K5hzcMPxc9IxPx2K4xO9l-oFiUw93daH3m5pluP6K7eOE6txBuRVfEcpJSwlelsOsW8gb8VJcnzMS9EnZpeA0tW_p-mnkFc3VcfyXuhe5R3G7aa5d8uHv70yJ9Y3-UhjiN9EhpMdfPAoEB9fYKKkJRzF7utTTIPGrSaSU6d2pcpfYKaxIwePzEkT4DfcQthoZdy9ucNvvLoi1DIC-UocFD8HLs8LYKEqSxQvOcvnThbObJ9af71EwmuE21fO5KzMW20KtAeget1gnldOosPtz1G5EwvaQ401-RPQzPGMVBld0_zMCAwZttJ4knw
        idp-certificate-authority: /root/ca.pem
        idp-issuer-url: https://oidcidp.tremolo.lan:8443/auth/idp/OidcIdP
        refresh-token: q1bKLFOyUiosTfawzA93TzZIDzH2TNa2SMm0zEiPKTUwME6BkEo6Sql5yUWVBSWpKUGphaWpxSVAfekBOZbBhaEW+VlFUeVRGcluyVF5JT4+haZmPsluFoFu5XkpXk5BXq
      name: oidc
```

`id_token`の有効期限が切れると、`kubectl`は`.kube/config`に格納してある`refresh_token`と`client_secret`を使って`id_token`を更新しようとします。

##### Option 2 - `--token`オプションを使う

`kubectl`コマンドには`--token`オプションを使ってトークンを渡せます。単純に`id_token`をこのオプションにコピーアンドペーストします。

```
kubectl --token=eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL21sYi50cmVtb2xvLmxhbjo4MDQzL2F1dGgvaWRwL29pZGMiLCJhdWQiOiJrdWJlcm5ldGVzIiwiZXhwIjoxNDc0NTk2NjY5LCJqdGkiOiI2RDUzNXoxUEpFNjJOR3QxaWVyYm9RIiwiaWF0IjoxNDc0NTk2MzY5LCJuYmYiOjE0NzQ1OTYyNDksInN1YiI6Im13aW5kdSIsInVzZXJfcm9sZSI6WyJ1c2VycyIsIm5ldy1uYW1lc3BhY2Utdmlld2VyIl0sImVtYWlsIjoibXdpbmR1QG5vbW9yZWplZGkuY29tIn0.f2As579n9VNoaKzoF-dOQGmXkFKf1FMyNV0-va_B63jn-_n9LGSCca_6IVMP8pO-Zb4KvRqGyTP0r3HkHxYy5c81AnIh8ijarruczl-TK_yF5akjSTHFZD-0gRzlevBDiH8Q79NAr-ky0P4iIXS8lY9Vnjch5MF74Zx0c3alKJHJUnnpjIACByfF2SCaYzbWFMUNat-K1PaUk5-ujMBG7yYnr95xD-63n8CO8teGUAAEMx6zRjzfhnhbzX-ajwZLGwGUBT4WqjMs70-6a7_8gZmLZb2az1cZynkFRj2BaCkVT3A2RrjeEwZEtGXlMqKJ1_I2ulrOVsYx01_yD35-rw get nodes
```

### Webhookトークン認証

Webhook認証は無記名トークンの認証に対するフックです。

* `--authentication-token-webhook-config-file`はリモートWebhookサービスへのアクセス方法を記述した構成ファイルです。
* `--authentication-token-webhook-cache-ttl`は認証結果をどれくらいキャッシュするかです。デフォルトは2分です。

構成ファイルには[kubeconfig](/ja/docs/concepts/cluster-administration/authenticate-across-clusters-kubeconfig/)ファイル形式を使います。ファイル内の`clusters`はリモートサービスを参照し、`users`はAPIサーバフックを参照します。例は次のようになります。

```yaml
# clustersはリモートサービスを参照します。
clusters:
  - name: name-of-remote-authn-service
    cluster:
      certificate-authority: /path/to/ca.pem         # リモートサービスを検証するためのCA
      server: https://authn.example.com/authenticate # クエリ発行先のリモートサービスのURL。'https'を使わなければなりません

# usersはAPIサーバのWebhook構成を参照します
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # cert for the webhook plugin to use
      client-key: /path/to/key.pem          # key matching the cert

# kubeconfigファイルにはcontextが必要です。APIサーバのためのものを提供します。
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authn-service
    user: name-of-api-sever
  name: webhook
```

クライアントが[上](#無記名トークンをリクエストに追加する)で議論したように無記名トークンを使ってAPIサーバを認証しようとすると、認証Webhookはトークンを含むJSONシリアル化された`authentication.k8s.io/v1beta` `TokenReview`オブジェクトをリモートサービスへPOSTします。Kubernetesはこのようなヘッダのないリクエストを疑いません。

Webhook APIオブジェクトは他のKubernetes APIオブジェクトと同じ[バージョン互換性ルール](/ja/docs/concepts/overview/kubernetes-api/)が適用されることに注意してください。実装者はベータオブジェクトに対しては互換性の保証がないことに注意し、正しいデシリアル化を保証するためにリクエストの"apiVersion"フィールドをチェックすべきです。加えて、APIサーバは`authentication.k8s.io/v1beta1` API拡張グループを有効にしなければなりません (`--runtime-config=authentication.k8s.io/v1beta1=true`)。

POSTボディは以下の形式になります。

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "spec": {
    "token": "(BEARERTOKEN)"
  }
}
```

リモートサービスによってログインの成功を示すためにリクエストの`status`フィールドが埋められることが期待されます。レスポンスボディの`spec`フィールドは無視されるので、省略されてもかまいません。無記名トークンの正常な検証結果は次のようになります。

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "janedoe@example.com",
      "uid": "42",
      "groups": [
        "developers",
        "qa"
      ],
      "extra": {
        "extrafield1": [
          "extravalue1",
          "extravalue2"
        ]
      }
    }
  }
}
```

失敗した場合は次のようになります。

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": false
  }
}
```

HTTPステータスコードは追加のエラーコンテキストを提供するために使われます。

### 認証プロキシ

APIサーバは`X-Remote-User`のようなリクエストヘッダの値からユーザを識別するように構成できます。これは認証プロキシとの組合せで使うために設計されています。

* `--requestheader-username-headers`は必須で、大文字小文字は区別されません。ユーザIDを順にチェックするためのヘッダ名です。値の含まれる最初のヘッダがユーザ名として使われます。
* `--requestheader-group-headers`は1.6以上で使えます。オプションで、大文字小文字は区別されません。"X-Remote-Group"が推奨されます。ユーザのグループを順にチェックするためのヘッダ名です。指定されたすべてのヘッダに含まれるすべての値がグループ名として使われます。
* `--requestheader-extra-headers-prefix`は1.6以上で使えます。オプションで、大文字小文字は区別されません。"X-Remote-Extra-"が推奨されます。ユーザについての追加情報 (通常は認証プラグインで使われる) を探すためのヘッダ接頭辞です。指定された接頭辞で始まるヘッダの接頭辞部分を削除します。ヘッダ名の残りは小文字に変換され、[パーセントデコード](https://tools.ietf.org/html/rfc3986#section-2.1)されてキーとなり、ヘッダの値が値となります。

{{< note >}}
**メモ:** 1.11.3より前では、キーは[HTTPで合法なヘッダラベル](https://tools.ietf.org/html/rfc7230#section-3.2.6)の文字のみを含められます。
{{< /note >}}

例えば、次のように構成すると、

```
--requestheader-username-headers=X-Remote-User
--requestheader-group-headers=X-Remote-Group
--requestheader-extra-headers-prefix=X-Remote-Extra-
```

次のリクエストは、

```http
GET / HTTP/1.1
X-Remote-User: fido
X-Remote-Group: dogs
X-Remote-Group: dachshunds
X-Remote-Extra-Acme.com%2Fproject: some-project
X-Remote-Extra-Scopes: openid
X-Remote-Extra-Scopes: profile
```

次のようなユーザ情報となる。

```yaml
name: fido
groups:
- dogs
- dachshunds
extra:
  acme.com/project:
  - some-project
  scopes:
  - openid
  - profile
```

ヘッダスプーフィングを防ぐため、認証プロキシはリクエストヘッダがチェックされる前に指定されたCAに対する検証のため、正当なクライアント証明書が必要となります。警告: CAの利用法を守るための仕組みとリスクを理解せずに、異なるコンテキストで使われるCAを再利用しないでください。

* `--requestheader-client-ca-file`は必須です。PEM符号化された証明書バンドルです。ユーザ名に対してリクエストヘッダがチェックされる前に、正当なクライアント証明書が提示され、指定されたファイルの認証局に対して検証されなければなりません。
* `--requestheader-allowed-names`はオプションです。Common Names (CN)のリストです。空であれば、どのようなCommon Nameでも許可されます。


## 匿名リクエスト

有効であれば、ほかに構成された認証メソッドで拒絶されなかったリクエストは匿名リクエストとして扱われ、`system:anonymous`のユーザ名と`system:unauthenticated`のグループ名が与えられます。

例えば、トークン認証が構成されたサーバで、匿名アクセスが有効になっていると、無効な無記名トークンを提供するリクエストは`401 Unauthorized`エラーを受けとります。トークンを提供しないリクエストは匿名リクエストとして扱われます。

1.5.1から1.5.xでは、匿名リクエストはデフォルトで無効になっており、APIサーバに`--anonymous-auth=true`を渡すことで有効にできます。

1.6以降、認証モードに`AlwaysAllow`以外が使われていれば、匿名アクセスはデフォルトで有効になっており、APIサーバに`--anonymous-auth=false`を渡すことで無効にできます。1.6から、ABACとRBAC認可モジュールは`system:anonymous`ユーザまたは`system:unauthenticated`グループの明示的な認可が必要なので、`*`ユーザまたは`*`グループへのアクセスを許可する古いポリシルールには匿名ユーザは含まれません。


## ユーザ切り替え

ユーザは切り替えヘッダを通じて他のユーザとして振る舞うことができます。これらはリクエストを認証するリクエストのユーザ情報に手動で上書きします。例えば、管理者は一時的に他のユーザに切り替えてリクエストが拒絶されるかどうかを見ることで、認可ポリシをデバッグするためにこの機能を使えます。

切り替えリクエストは最初にリクエストしたユーザとして認証し、その後切り替え先のユーザ情報に入れ替わります。

* ユーザは資格情報 _と_ 切り替えヘッダでAPI呼び出しを作る。
* APIサーバはユーザを認証する。
* APIサーバはユーザが切り替え権限を持っているかどうか確認する。
* リクエストユーザ情報を切り替え値で置き換える。
* リクエストが評価され、認可は切り替えられたユーザ情報で行われる。

以下のHTTPヘッダが切り替えリクエストの実行に使われる。

* `Impersonate-User`: 切り替え先のユーザ名
* `Impersonate-Group`: 切り替え先のグループ名。複数グループを設定するために複数回指定できる。オプション。"Impersonate-User"が必要
* `Impersonate-Extra-( extra name )`: 追加フィールドをユーザに関連付けるための動的ヘッダ。オプション。"Impersonate-User"が必要。一貫性を保つために、`( extra name )`は小文字であるべきで、[HTTPで合法なヘッダラベル](https://tools.ietf.org/html/rfc7230#section-3.2.6)でない文字はUTF8で[パーセントエンコード](https://tools.ietf.org/html/rfc3986#section-2.1)されなければなりません。

{{< note >}}
**メモ:** 1.11.3 (と1.10.7, 1.9.11) より前では、`( extra name )`は[HTTPで合法なヘッダラベル](https://tools.ietf.org/html/rfc7230#section-3.2.6)のみを含むことができます。
{{< /note >}}

ヘッダのセットの例:

```http
Impersonate-User: jane.doe@example.com
Impersonate-Group: developers
Impersonate-Group: admins
Impersonate-Extra-dn: cn=jane,ou=engineers,dc=example,dc=com
Impersonate-Extra-acme.com%2Fproject: some-project
Impersonate-Extra-scopes: view
Impersonate-Extra-scopes: development
```

`kubectl`を使うことで`Impersonate-User`ヘッダを構成するために`--as`フラグを設定する時、`Impersonate-Group`ヘッダを構成するために`--as-group`フラグを使います。

```shell
$ kubectl drain mynode
Error from server (Forbidden): User "clark" cannot get nodes at the cluster scope. (get nodes mynode)

$ kubectl drain mynode --as=superman --as-group=system:masters
node/mynode cordoned
node/mynode drained
```

ユーザやグループを切り替えたり追加フィールドを設定するために、切り替えユーザは切り替える属性 ("user"や"group"など) の種類の"impersonate" verbを実行する能力を持たなければなりません。RBAC認可プラグインが有効なクラスタに対して、以下のClusterRoleはユーザとグループの切り替えヘッダをセットするのに必要なルールを包含します。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
```

追加フィールドはリソース "userextras"のサブリソースとして評価されます。ユーザに追加フィールド"scope"に対する切り替えヘッダを使う許可を与えるためには、ユーザは以下のロールの権限が必要です。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: scopes-impersonator
rules:
# "Impersonate-Extra-scopes"ヘッダを設定できます
- apiGroups: ["authentication.k8s.io"]
  resources: ["userextras/scopes"]
  verbs: ["impersonate"]
```

切り替えヘッダの値は、リソースが取ることのできる`resourceNames`のセットに限定することで制限することもできます。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: limited-impersonator
rules:
# ユーザ"jane.doe@example.com"に切り替えられます
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
  resourceNames: ["jane.doe@example.com"]

# グループ"developers"と"admins"に切り替えられます
- apiGroups: [""]
  resources: ["groups"]
  verbs: ["impersonate"]
  resourceNames: ["developers","admins"]

# "view"と"development"の値で追加フィールド"scope"に切り替えられます
- apiGroups: ["authentication.k8s.io"]
  resources: ["userextras/scopes"]
  verbs: ["impersonate"]
  resourceNames: ["view", "development"]
```

## client-go 資格情報プラグイン

{{< feature-state for_k8s_version="v1.11" state="beta" >}}

`kubelet`や`kubectl`のような`k8s.io/client-go`とそれを使うツールはユーザ資格情報を受け取るために外部コマンドを実行できます。

この機能は`k8s.io/client-go`によってネイティブにサポートされない認証プロトコル (LDAP, Kerberos, OAuth2, SAMLなど) とのクライアントサイド統合を意図したものです。このプラグインはプロトコル特有のロジックを実行し、不透明な資格情報を返します。ほぼすべての資格情報プラグインユースケースはクライアントプラグインによって生成された資格情報形式を解釈するために[Webhookトークン認証プラグイン](#webhookトークン認証プラグイン)をサポートするサーバサイドコンポーネントを必要とします。

### ユースケース例

仮説に基づいたユースケースでは、組織はユーザ固有のLDAP資格情報を交換する外部サービスを実行しています。このサービスはトークンを検証するための[Webhookトークン認証プラグイン](#webhookトークン認証プラグイン)リクエストにレスポンスを返す機能もあります。ユーザはワークステーションに資格情報プラグインをインストールする必要があります。

APIに対して認証するために、

* ユーザは`kubectl`コマンドを発行する
* 資格情報プラグインはユーザにLDAP資格情報に対するプロンプトを出し、トークンのための外部サービスと資格情報を交換する
* 資格情報プラグインはclient-goにトークンを返し、client-goはAPIサーバに対する無記名トークンとして使う
* APIサーバは外部サービスに`TokenReview`を送信するために[Webhookトークン認証プラグイン](#webhookトークン認証プラグイン)を使う
* 外部サービスはトークンの署名を検証し、ユーザのユーザ名とパスワードを返す

### 構成

資格情報プラグインは[kubectl構成ファイル](/ja/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)のuserフィールドの一部を通じて構成されます。

```yaml
apiVersion: v1
kind: Config
users:
- name: my-user
  user:
    exec:
      # 実行するコマンド。必須
      command: "example-client-go-exec-plugin"

      # ExecCredentialsリソースをデコードする時に使うAPIバージョン。必須
      #
      # プラグインによって返されるAPIバージョンはここで挙げたバージョンとマッチしなければならない
      #
      # (client.authentication.k8s.io/v1alpha1のような) 複数のバージョンをサポートするツールと統合するため、
      # 実行プラグインが期待するバージョンを指示する環境変数を設定するか、ツールに引数を渡す
      apiVersion: "client.authentication.k8s.io/v1beta1"

      # プラグインを実行する時に設定する環境変数。オプション
      env:
      - name: "FOO"
        value: "bar"

      # プラグインを実行する時に渡す引数。オプション
      args:
      - "arg1"
      - "arg2"
clusters:
- name: my-cluster
  cluster:
    server: "https://172.17.4.100:6443"
    certificate-authority: "/etc/kubernetes/ca.pem"
contexts:
- name: my-cluster
  context:
    cluster: my-cluster
    user: my-user
current-context: my-cluster
```

相対コマンドパスは構成ファイルのディレクトリからの相対として解釈されます。もしKUBECONFIGが`/home/jane/kubeconfig`に設定されていて、実行コマンドが`./bin/example-client-go-exec-plugin`であれば、バイナリ`/home/jane/bin/example-client-go-exec-plugin`が実行されます。

```yaml
- name: my-user
  user:
    exec:
      # kubeconfigのディレクトリからの相対パス
      command: "./bin/example-client-go-exec-plugin"
      apiVersion: "client.authentication.k8s.io/v1beta1"
```

### 入力フォーマットと出力フォーマット

実行されたコマンドは`ExecCredential`オブジェクトを`stdout`に出力します。`k8s.io/client-go`は`status`にある返された資格情報を使ってKubernetes APIに対して認証します。

インタラクティブセッションから実行する時、`stdin`はプラグインに直接公開されます。プラグインはユーザにインタラクティブに表示するかどうか決めるために[TTYチェック](https://godoc.org/golang.org/x/crypto/ssh/terminal#IsTerminal)を使うべきです。

無記名トークン資格情報を使うために、プラグインは`ExecCredential`のステータス内にトークンを返します。

```json
{
  "apiVersion": "client.authentication.k8s.io/v1beta1",
  "kind": "ExecCredential",
  "status": {
    "token": "my-bearer-token"
  }
}
```

別の方法として、PEM符号化されたクライアント証明書とキーがTLSクライアント認証を使うために返されます。プラグインが後段の呼びだしで異なる証明書とキーを返すのなら、`k8s.io/client-go`は新しいTLSハンドシェイクを強制するためにサーバとの既存の接続をクローズします。

指定されれば、`clientKeyData`と`clientCertificateData`は両方とも存在しなければなりません。

`ckuentCertificateData`はサーバに送信する追加の中間証明書を含むことができます。

```json
{
  "apiVersion": "client.authentication.k8s.io/v1beta1",
  "kind": "ExecCredential",
  "status": {
    "clientCertificateData": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "clientKeyData": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
  }
}
```

任意で、レスポンスはRFC3339タイムスタンプでフォーマットされた資格情報の有効期限を含められます。有効期限が存在するかしないかで以下のような影響があります。

- 有効期限が含まれていれば、無記名トークンとTLS資格情報は有効期限に達するか、サーバが401 HTTPステータスコードを返すか、プロセスが終了するまでキャッシュされます
- 有効期限が省略されれば、無記名トークンとTLS資格情報はサーバが401 HTTPステータスコードを返すかプロセスが終了するまでキャッシュされます

```json
{
  "apiVersion": "client.authentication.k8s.io/v1beta1",
  "kind": "ExecCredential",
  "status": {
    "token": "my-bearer-token",
    "expirationTimestamp": "2018-03-05T17:30:20-08:00"
  }
}
```
{{% /capture %}}
