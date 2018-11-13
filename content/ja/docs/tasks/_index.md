---
title: タスク
main_menu: true
weight: 50
content_template: templates/concept
---

{{< toc >}}

{{% capture overview %}}

このセクションには、個々のタスクをどのように実行するのかを示すページがあります。タスクページでは
1つのことを実行する方法を、短いステップを通じて示します。

{{% /capture %}}

{{% capture body %}}

## Web UI (Dashboard)

Kubernetesクラスタでのコンテナ化されたアプリケーションの管理と監視を助けるDashboard Webユーザ
インタフェースをデプロイし、アクセスします。

## kubectlコマンドラインの使用 {#using-the-kubectl-command-line}

Kubernetesクラスタを直接管理するために使う、`kubectl`コマンドラインツールをインストールし、
セットアップします。

## Podとコンテナの構成 {#configuring-pods-and-containers}

Podとコンテナの一般的な構成タスクを実行します。

## アプリケーションの実行 {#running-applications}

ローリングアップデートやPodへの情報の注入、水平Podオートスケーリングといった、一般的な
アプリケーションの管理タスクを実行します。

## Jobの実行 {#running-jobs}

並列実行を使ってJobを実行します。

## クラスタ内のアプリケーションへのアクセス {#accessing-applications-in-a-cluster}

クラスタ内のアプリケーションにアクセスするために、ロードバランシングやポートフォワーディングを
構成したり、ファイアウォールやDNS構成をセットアップします。

## 監視、ログ、デバッグ {#monitoring-logging-and-debugging}

クラスタをトラブルシュートするための監視やログ、さらにコンテナ化されたアプリケーションのデバッグを
セットアップします。

## Kubernetes APIへのアクセス {#accessing-the-kubernetes-api}

Kubernetes APIに直接アクセスするさまざまな方法を学びます。

## TLSの使用 {#using-tls}

クラスタのルート証明局 (CA) を信頼し、使用するようアプリケーションを構成します。

## クラスタの管理 {#administering-a-cluster}

クラスタを管理する一般的なタスクを学びます。

## 連合の管理 {#administering-federation}

クラスタ連合のコンポーネントを構成します。

## ステートフルアプリケーションの管理 {#managing-stateful-application}

スケーリングや削除、StatefulSetのデバッグを含む、ステートフルアプリケーションを管理するための
一般的なタスクを実行します。

## クラスタデーモン {#cluster-daemons}

ローリングアップデートの実行といった、DaemonSetを管理するための一般的なタスクを実行します。

## GPUの管理 {#managing-gpus}

ノードのリソースとして使われるNVIDIA GPUを構成し、スケジュールします。

## HugePageの管理 {#managing-hugepages}

クラスタのスケジュール可能なリソースとして、HugePageを構成し、スケジュールします。

{{% /capture %}}

{{% capture whatsnext %}}

タスクページを執筆したいのであれば、
[Creating a Documentation Pull Request](/docs/home/contribute/create-pull-request/)を
参照してください。

{{% /capture %}}
