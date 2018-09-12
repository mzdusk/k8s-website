---
title: Ingress
content_template: templates/concept
weight: 40
---

{{% capture overview %}}
{{< glossary_definition term_id="ingress" length="all" >}}
{{% /capture %}}

{{% capture body %}}
## 用語

* ノード: Kubernetesクラスター内の単一の仮想または物理マシン。
* クラスタ: インターネットからファイアウォールで守られたノードのグループで、Kubernetesにより管理される主要な計算資源です。
* エッジルータ: クラスタに対しファイアウォールポリシーを適用するルータ。クラウドプロバイダやハードウェアの一部によって管理されるゲートウェイの場合もあります。
* クラスタネットワーク: [Kubernetes ネットワーキングモデル](/ja/docs/concepts/cluster-administration/networking/)によってクラスタ内での通信を円滑にする物理的または論理的なリンクの集合。クラスタネットワークの例には [flannel](https://github.com/coreos/flannel#flannel) のような Overlays や [OVS](https://www.openvswitch.org/) のような SDN があります。
* サービス: ラベルセレクタを使ってPodの集合を識別する Kubernetes [Service](/ja/docs/concepts/services-networking/service/)。特に言及がなければ、Service はクラスタネットワーク内でのみルーティング可能な仮想IPを持ちます。

## Ingress とは何か？

通常、サービスと Pod はクラスタネットワークによってのみルーティング可能な IP を持ちます。エッジルータに到達したトラフィックはドロップされるかどこかに転送されます。概念的にはこのような感じです。

```none
    internet
        |
  ------------
  [ Services ]
```

Ingress は内向きの接続をクラスタサービスに到達させることを許可するルールの集合です。

```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

外部到達可能な URL の付与や、トラフィックのロードバランス、SSL の終端、名前ベースのバーチャルホストの提供などをサービスに構成させられます。ユーザは API サーバに Ingress リソースを POST することで Ingress を要求できます。[Ingress コントローラ](#ingress-コントローラ)は通常ロードバランサで Ingress を実行する責任を負いますが、エッジルータや HA な方法でトラフィックを扱う追加のフロントエンドを構成することもできます。

## 前提条件

Ingress リソースを使い始める前に、いくつかの事柄を理解しておくべきです。Ingress は Beta リソースで Kubernetes 1.1 以前のリリースでは利用できません。Ingress を満たすために Ingress コントローラが必要で、単純にリソースを作成するだけでは効果はありません。

GCE/Google Kubernetes Engine はマスタに Ingress コントローラをデプロイします。Pod でいくつでもカスタム Ingress コントローラをデプロイできます。[ここ](https://git.k8s.io/ingress#running-multiple-ingress-controllers)と[ここ](https://git.k8s.io/ingress-gce/BETA_LIMITATIONS.md#disabling-glbc)で指示されているように、各 Ingress には適切なクラスをアノテーションしなければなりません。

このコントローラの [Beta 制限](https://github.com/kubernetes/ingress-gce/blob/master/BETA_LIMITATIONS.md#glbc-beta-limitations) を確認しておいて下さい。GCE/Google Kubernetes Engine 以外の環境では Pod として[コントローラをデプロイ](https://git.k8s.io/ingress-nginx/README.md)する必要があります。

## Ingress リソース

最低限の Ingress は次にようになります。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        backend:
          serviceName: test
          servicePort: 80
```

*Ingress コントローラを構成していなければ、これを API サーバに POST しても効果はありません。

__1-6行目__: 他のすべての Kubernetes 構成と同じく、Ingress は `apiVersion`, `kind`, `metadata`フィールドが必要です。構成ファイルの利用についての一般的な情報は、[アプリケーションのデプロイ](/ja/docs/tasks/run-application/run-stateless-application-deployment/), [コンテナの構成](/ja/docs/tasks/configure-pod-container/configure-pod-configmap/), [リソースの管理](/ja/docs/concepts/cluster-administration/manage-deployment/), [Ingress 構成の書き換え](https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/rewrite/README.md)を参照してください。

__7-9行目__: Ingress [スペック](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status)はロードバランサやプロキシサーバを構成するのに必要な全ての情報を持ちます。最も重要なのは、全ての内向きリクエストに対してマッチするルールのリストを含むことです。現在、Ingress リソースは http ルールのみをサポートします。

__10-11行目__: 各 http ルールには次の情報が含まれます。ホスト (例えば foo.bar.com で、デフォルトは *)、それぞれに関連するバックエンド (test:80) を持つパス (例えば /testpath) のリスト。ホストとパスは両方ともロードバランサがトラフィックをバックエンドに振り分ける前の内向きリクエストの内容にマッチしなければなりません。

__12-14行目__: バックエンドは [Service ドキュメント](/docs/concepts/services-networking/service/)に記述されているような service:port の組み合わせです。Ingress トラフィックは通常、マッチするバックエンドのエンドポイントに直接送信されます。

__グローバルパラメータ__: 簡単のため、例のIngressにはグローバルパラメータがないため、リソースの完全な定義は [API リファレンス](https://releases.k8s.io/{{< param "githubbranch" >}}/staging/src/k8s.io/api/extensions/v1beta1/types.go)を参照してください。これには、スペックのパスにマッチしないリクエストが Ingress コントローラのデフォルトバックエンドに送られた場合のグローバルなデフォルトバックエンドを指定できます。

## Ingress コントローラ

Ingress リソースが動作するために、クラスタは起動している Ingress コントローラが必要です。これは `kube-controller-manager` の一部として動作し、クラスタ作成の際に自動的に起動する他のコントローラとは異なります。クラスタに最もよく適合する Ingress コントローラ実装を選ぶか、新しい Ingress コントローラを実装するかします。

* Kubernetes は現在、[GCE](https://git.k8s.io/ingress-gce/README.md) と [nginx](https://git.k8s.io/ingress-nginx/README.md) コントローラのサポートとメンテナンスをしています。
* F5 Networks は [F5 BIG-IP Controller for Kubernetes](http://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/latest) の[サポートとメンテナンス](https://support.f5.com/csp/article/K86859508)をしています。
* [Kong](https://konghq.com) は [Kong Ingress Controller for Kubernetes](https://konghq.com/blog/kubernetes-ingress-controller-for-kong/) の[コミュニティ](https://discuss.konghq.com/c/kubernetes)と[商用](https://konghq.com/api-customer-success/)のサポートとメンテナンスを提供しています。
* [Traefik](https://github.com/containous/traefik)はフル機能の Ingress コントローラ ([Let's Encrypt](https://letsencrypt.org), secrets, http2, websocket)で、[Containous](https://containo.us/services) による商用サポートも受けられます。

## Ingress のタイプ

### 単一サービス Ingress

単一の Service を公開する Kubernetes の概念は存在しますが、*default backend* にルールを指定しないことで Ingress を通じても行うことができます。

{{< code file="ingress.yaml" >}}

`kubectl create -f` を使って作成すると、状態を見ることができます。

```shell
$ kubectl get ing
NAME                RULE          BACKEND        ADDRESS
test-ingress        -             testsvc:80     107.178.254.228
```

ここで、`107.178.254.228` はこの Ingress を満たす Ingress コントローラによって割り当てられた IP です。`RULE` 列はこの IP に送られるすべてのトラフィックが `BACKEND` に列挙された Kubernetes Service に振り分けられることを示しています。

### シンプルな fanout

以前述べたように、kubernetes 内の Pod はクラスタネットワークからのみ見える IP を持っているので、エッジで Ingress トラフィックを受け入れ、正しいエンドポイントにプロキシする何かが必要です。このコンポーネントは通常、高可用なロードバランサです。Ingress はロードバランサ数を維持できるようにします。例えば次のようにセットアップする場合

```shell
foo.bar.com -> 178.91.123.132 -> / foo    s1:80
                                 / bar    s2:80
```

Ingress は次のようになります。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

`kubectl create -f` で作成すると次のような状態になります。

```shell
$ kubectl get ing
NAME      RULE          BACKEND   ADDRESS
test      -
          foo.bar.com
          /foo          s1:80
          /bar          s2:80
```

Ingress コントローラは、Service (s1, s2) が存在する限り、Ingress を満たす特定のロードバランサの実装を供給します。これが完了すると、Ingress の最終列に下にロードバランサのアドレスが見えるようになります。

### 名前ベースのバーチャルホスト

名前ベースのバーチャルホストは同じIPアドレスに対して複数のホスト名を使います。

```none
foo.bar.com --|                 |-> foo.bar.com s1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com s2:80
```

下記のIngressは裏のロードバランサに[Hostヘッダ](https://tools.ietf.org/html/rfc7230#section-5.4)に基づいたリクエストのルーティングをするよう伝えます。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80
```

__デフォルトバックエンド__: 前のセクションで示したもののようなルールのない Ingress はすべてのトラフィックを単一のデフォルトバックエンドへ送ります。ルールとデフォルトバックエンドの集合を指定することで、ウェブサイトの 404 ページの場所をロードバランサに伝えるために同じテクニックを使うことができます。リクエストヘッダの Host に Ingress の Host がマッチしないか、リクエストの URL がパスにマッチしなければ、トラフィックはデフォルトバックエンドへルーティングされます。

### TLS

TLS秘密鍵と証明書を含む [Secret](/docs/user-guide/secrets) を指定することで Ingress をセキュアにできます。現在は単一の TLS ポート 443 のみをサポートし、TLS の終端であるとみなします。Ingress の TLS 構成セクションで異なるホストを指定すると、(SNI をサポートする Ingress コントローラによって提供される) SNI TLS 拡張を通じて指定されたホスト名によって同じポートが多重化されます。TLS シークレットには、TLS で使う証明書と秘密鍵を含む `tls.crt` と `tls.key` という名前のキーを含めなければなりません。

```yaml
apiVersion: v1
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
kind: Secret
metadata:
  name: testsecret
  namespace: default
type: Opaque
```

Ingress でこの Secret を参照することで、Ingress コントローラにクライアントからロードバランサまでの通信路を TLS を使ってセキュアにするよう伝えます。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: no-rules-map
spec:
  tls:
  - secretName: testsecret
  backend:
    serviceName: s1
    servicePort: 80
```

さまざまな Ingress コントローラごとにサポートされる TLS の機能に違いがあることに注意してください。TLS がどのように動作するのか理解するために、[nginx](https://git.k8s.io/ingress-nginx/README.md#https) や [GCE](https://git.k8s.io/ingress-gce/README.md#frontend-https) など、指定した Ingress コントローラのドキュメントを参照してください。

### ロードバランシング

Ingress コントローラは、ロードバランシングアルゴリズムやバックエンドウェイトスキームなどといったすべての Ingress に適用するいくつかのロードバランシングポリシ設定とともにブートします。更に高度なロードバランシングコンセプト (永続セッションや動的ウェイトなど) はまだ Ingress を通じて公開されていません。これらの機能は [Service ロードバランサ](https://github.com/kubernetes/ingress-nginx/blob/master/docs/ingress-controller-catalog.md)を通じて得られます。経時的に、クロスプラットフォームで Ingress リソースに適用可能なロードバランシングパターンの抽出を計画しています。

ヘルスチェックが Ingress を通じて直接公開されていなくても、最終的には同じ目的を達成させることのできる [readiness probe](/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) のようなコンセプトが存在することは覚えておく価値があります。コントローラがどのようにヘルスチェックを扱っているのかを知るためにドキュメントを見直してください ([nginx](https://git.k8s.io/ingress-nginx/README.md), [GCE](https://git.k8s.io/ingress-gce/README.md#health-checks))。

## Ingress の更新

既存の Ingress に新しい Host を追加する場合、リソースを編集することで更新できます。

```shell
$ kubectl get ing
NAME      RULE          BACKEND   ADDRESS
test      -                       178.91.123.132
          foo.bar.com
          /foo          s1:80
$ kubectl edit ing test
```

既存のYAMLのエディタが起動するので、新しいHostを含むよう編集します。

```yaml
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
        path: /foo
  - host: bar.baz.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80
        path: /foo
..
```

この YAML を保存することで API サーバのリソースが更新され、Ingress コントローラにロードバランサを再構成するよう伝えられます。

```shell
$ kubectl get ing
NAME      RULE          BACKEND   ADDRESS
test      -                       178.91.123.132
          foo.bar.com
          /foo          s1:80
          bar.baz.com
          /foo          s2:80
```

編集した Ingress の YAML ファイルに `kubectl replace -f` を実行することで同じことが行えます。

## アベイラビリティゾーンをまたぐ失敗

失敗ドメインをまたいで拡散するトラフィックに対するテクニックはクラウドプロバイダ間で異なります。詳細は関係する Ingress コントローラのドキュメントをチェックしてください。連合クラスタに Ingress をデプロイする詳細は連合の[ドキュメント](/ja/docs/concepts/cluster-administration/federation/)を参照してください。

## 今後の取り組み

* HTTPS/TLS のさまざまなモードのサポート (例えば SNI や再暗号化)
* Claim を通じた IP またはホスト名のリクエスト
* L4 と L7 Ingress の結合
* さらなる Ingress コントローラ

このリソースの進化についての詳細は [L7 and Ingress proposal](https://github.com/kubernetes/kubernetes/pull/12827) を、さまざまな Ingress コントローラの進化についての詳細は [Ingressリポジトリ](https://github.com/kubernetes/ingress/tree/master) を参照してください。

## 代替手段

Service は Ingress リソースを直接使わない複数の方法で公開できる。

* [Service.Type=LoadBalancer](/ja/docs/concepts/services-networking/service/#type-loadbalancer) を使う
* [Service.Type=NodePort](/docs/concepts/services-networking/service/#type-nodeport) を使う
* [Portプロキシ](https://git.k8s.io/contrib/for-demos/proxy-to-service) を使う
{{% /capture %}}

{{% capture whatsnext %}}

{{% /capture %}}
