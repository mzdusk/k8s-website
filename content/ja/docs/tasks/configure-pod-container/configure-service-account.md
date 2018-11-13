---
title: Podに対するサービスアカウントの構成
content_template: templates/task
weight: 90
---

{{% capture overview %}}
ServiceAccountはPodで実行するプロセスのIDを提供します。

*これはユーザ向けのサービスアカウントの紹介です。
[サービスアカウントのクラスタ管理者向けガイド](/docs/reference/access-authn-authz/service-accounts-admin/)も
参照してください。*

{{< note >}}
**メモ:** このドキュメントは、Kubernetesプロジェクトが推奨する形でセットアップされたクラスタ内で
サービスアカウントがどのようにふるまうかについて書かれています。クラスタ管理者がそのふるまいを
カスタマイズしている場合、このドキュメントは適用できません。
{{< /note >}}

人間が (例えば`kubectl`を使って) クラスタにアクセスする場合、その人はあるユーザアカウント
(クラスタ管理者がクラスタをカスタマイズしていなければ、`admin`となります) として、
APIサーバに認証されます。Podで実行するコンテナのプロセスもAPIサーバにアクセスできます。
そうすると、それらはあるサービスアカウント (例えば`default`) として認証されます。

{{% /capture %}}

{{< toc >}}

{{% capture prerequisites %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

{{% /capture %}}

{{% capture steps %}}

## APIサーバにアクセスするためにデフォルトサービスアカウントを使う

Podを作成すると、サービスアカウントを指定しなければ、同じ名前空間の`default`サービスアカウントが
自動的に割り当てられます。作成したPodの生JSONやYAMLを取得すると (例えば、`kubectl get pods/podname -o yaml`)、
`spec.serviceAccountName`フィールドが
[自動的に設定](/docs/user-guide/working-with-resources/#resources-are-automatically-modified)
されていることが分かります。

[Accessing the Cluster](/docs/user-guide/accessing-the-cluster/#accessing-the-api-from-a-pod)に
書かれているように、自動的にマウントされたサービスアカウント資格情報を使って、Pod内からAPIにアクセス
できます。そのサービスアカウントのAPIパーミッションは、使われている
[認可プラグインとポリシ](/docs/reference/access-authn-authz/authorization/#authorization-modules)
に依存します。

バージョン1.6から、サービスアカウントの`automountServiceAccountToken: false`を設定することで、
そのサービスアカウントの自動マウントを無効にできます。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```

バージョン1.6から、特定のPodに対して、API資格情報の自動マウントを無効にするようにもできます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  ...
```

両方が `automountServiceAccountToken`の値を指定していれば、Podスペックは先に来たものを
採用します。

## 複数のServiceAccountを使う

すべての名前空間には`default`と呼ばれるデフォルトのサービスアカウントリソースがあります。
名前空間のserviceAccountリソースは次のコマンドで列挙できます。

```shell
kubectl get serviceAccounts
NAME      SECRETS    AGE
default   1          1d
```

次のようにして、ServiceAccountオブジェクトを追加することができます。

```shell
kubectl create -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF
serviceaccount/build-robot created
```

次のようにして、サービスアカウントオブジェクトの完全なダンプを取得できます。

```shell
kubectl get serviceaccounts/build-robot -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-06-16T00:12:59Z
  name: build-robot
  namespace: default
  resourceVersion: "272500"
  selfLink: /api/v1/namespaces/default/serviceaccounts/build-robot
  uid: 721ab723-13bc-11e5-aec2-42010af0021e
secrets:
- name: build-robot-token-bvbk5
```

トークンが自動的に作成され、そのサービスアカウントから参照されていることがわかります。

[ServiceAccountにパーミッションを設定する](/docs/reference/access-authn-authz/rbac/#service-account-permissions)
ために、認可プラグインを使ってもかまいません。

デフォルトでないサービスアカウントを使うためには、単純にPodの`spec.serviceAccountName`に
使いたいサービスアカウントの名前をを設定します。

そのサービスアカウントはPodが作成される時に存在している必要があり、そうでなければ拒絶されます。

作成済みのPodのサービスアカウントは更新できません。

この例で作成したサービスアカウントは次のようにしてクリーンアップできます。

```shell
kubectl delete serviceaccount/build-robot
```

## ServiceAccount APIトークンを手動で作成する

上の出てきたような"build-robot"というサービスアカウントがあり、手動で新しいSecretを作成するとします。

```shell
kubectl create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-robot
type: kubernetes.io/service-account-token
EOF
secret/build-robot-secret created
```

新しく作成されたSecretが"build-robot"サービスアカウントにAPIトークンを投入しているか確認します。

存在しないサービスアカウントに対するトークンは、トークンコントローラによってクリーンアップされます。

```shell
kubectl describe secrets/build-robot-secret
Name:           build-robot-secret
Namespace:      default
Labels:         <none>
Annotations:    kubernetes.io/service-account.name=build-robot
                kubernetes.io/service-account.uid=da68f9c6-9d26-11e7-b84e-002dc52800da

Type:   kubernetes.io/service-account-token

Data
====
ca.crt:         1338 bytes
namespace:      7 bytes
token:          ...
```

{{< note >}}
**メモ:** `token`の内容はここでは省略します。
{{< /note >}}

## ServiceAccountにImagePullSecretを追加する

最初に、[here](/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod)で
説明されているように、imagePullSecretを作成します。次に、作成されたことを確認します。

```shell
kubectl get secrets myregistrykey
NAME             TYPE                              DATA    AGE
myregistrykey    kubernetes.io/.dockerconfigjson   1       1d
```

次に、このSecretをimagePullSecretとして使うよう、デフォルトサービスアカウントを編集します。

```shell
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "myregistrykey"}]}'
```

インタラクティブに行う場合は、次のようにします。

```shell
kubectl get serviceaccounts default -o yaml > ./sa.yaml

cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  resourceVersion: "243024"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge

vi sa.yaml
[editor session not shown]
[delete line with key "resourceVersion"]
[add lines with "imagePullSecrets:"]

cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge
imagePullSecrets:
- name: myregistrykey

kubectl replace serviceaccount default -f ./sa.yaml
serviceaccounts/default
```

これで、この名前空間で新規に作成されるPodは、スペックに以下が追加されます。

```yaml
spec:
  imagePullSecrets:
  - name: myregistrykey
```

## ServiceAccountTokenVolumeProjection

{{< feature-state for_k8s_version="v1.12" state="beta" >}}

{{< note >}}
**メモ:** 1.12では、このServiceAccountTokenVolumeProjectionは __ベータ__ で、
APIサーバに以下のフラグをすべて渡して有効にします。

* `--service-account-issuer`
* `--service-account-signing-key-file`
* `--service-account-api-audiences`

{{< /note >}}

kubeletはPodにサービスアカウントトークンを提示することもできます。audienceや
有効期間といった、希望するトークンのプロパティを指定できます。これらのプロパティは、
デフォルトサービスアカウントトークンには構成できません。このサービスアカウントトークンは
Podやサービスアカウントが削除された場合に無効にすることもできます。

このふるまいは[ServiceAccountToken](/docs/concepts/storage/volumes/#projected)と呼ばれる
ProjectedVolumeタイプを使って、PodSpecに構成されます。"vault"というaudienceと
2時間の有効期限のトークンをPodに提供するためには、次のPodSpecで構成します。

```yaml
kind: Pod
apiVersion: v1
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 7200
          audience: vault
```

kubeletはPodの代わりにトークンの要求と保存を行い、構成可能なファイルパスでトークンを利用可能にし、
有効期限が過ぎるとトークンを更新します。kubeletは、トークンが生存期間の80%を過ぎるか、24時間
過ぎると、トークンを入れ替えます。

アプリケーションは、トークンが入れ替わった場合に、再読み込みする責任を負います。ほとんどの場合、
定期的な再読み込み (5分ごとなど) で十分です。

{{% /capture %}}
