---
title: Podの概要
content_template: templates/concept
weight: 10
---

{{% capture overview %}}
このページでは、Kubernetesオブジェクトモデルでのデプロイ可能な最小オブジェクトである `Pod`の概要について述べます。
{{% /capture %}}

{{< toc >}}

{{% capture body %}}
## Podについて {#understanging-pods}

*Pod* はKubernetesで基本の構成要素で、作成するKubernetesオブジェクトモデルでの最も小さく、最もシンプルな単位です。Podはクラスタで実行するプロセスを表現します。

Podはアプリケーションコンテナ (複数の場合もある) やストレージリソース、一意なネットワークIP、どのようにコンテナを実行するのかを制御するオブションをカプセル化します。Podはデプロイの単位を表現します。 *Kubernetesでの単一のアプリケーションインスタンス* で、単一のコンテナまたは、密に結合しリソースを共有する少数のコンテナで構成されます。

> [Docker](https://www.docker.com)はKubernetes Podで使われる最も一般的なコンテナランタイムですが、他のコンテナランタイムも同じようにサポートします。

KubernetesクラスタのPodは主に2つの方法で使われます。

* **単一コンテナを実行するPod**: "1コンテナPod"モデルは最も一般的なKubernetesのユースケースです。この場合、Podは単一コンテナのラッパとして考えればよく、Kuberentesはコンテナを直接ではなくPodを管理します。
* **一緒に動作する必要のある複数のコンテナを実行するPod**: Podは密に結合し、リソースを共有する必要のある複数のコンテナで構成されるアプリケーションをカプセル化できます。これらのコンテナはサービスの集合体を形成します。例えば、1つのコンテナが共有ボリュームからファイルを公開し、分割した"サイドカー"コンテナがそれらのファイルを更新するような構成が考えられます。Podは単一の管理可能なエンティティとしてこれらのコンテナとストレージリソースを一緒にラップします。

[Kubernetes Blog](http://blog.kubernetes.io)にはPodのユースケースについての追加情報がいくつかあります。詳細は以下を参照してください。

* [The Distributed System Toolkit: Patterns for Composite Containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns)
* [Container Design Patterns](https://kubernetes.io/blog/2016/06/container-design-patterns)


各Podはアプリケーションの単一インスタンスを実行するよう意図されています。アプリケーションを水平にスケールさせたい場合、複数のPodを使うべきです。Kubernetesでは、これを _複製 (レプリケーション)_ と呼んでいます。複製されたPodは通常、コントローラと呼ばれる抽象化されたグループとして作成・管理されます。詳細は[Podとコントローラ](#pods-and-controllers)を参照してください。

### Podが複数のコンテナを管理する方法 {#how-pods-manage-multiple-containers}

Podはサービスの集合体を形成する複数の (コンテナとして) 協働するプロセスをサポートするよう設計されています。Podの中のコンテナは、自動的に同じ物理/仮想マシンに配置されます。コンテナはリソースと依存関係を共有でき、おたがいと通信でき、終了する時間と方法を連携できます。

単一のPodの中に複数のコンテナをグルーピングすることは比較的行動なユースケースであることに注意してください。このパターンは密に結合したコンテナがある特定のインスタンスに限定して使うべきです。例えば、下の図のような、共有ボリュームのファイルに対するWebブラウザとしてふるまうコンテナと、リモートからそれらのファイルを更新する分離された「サイドカー」コンテナのような構成です。

{{< figure src="/images/docs/pod.svg" title="pod diagram" width="50%" >}}

Podはそれを構成するコンテナに2種類の共有リソースを提供します。 *ネットワーク* と *ストレージ* です。

#### ネットワーク {#networking}

各Podには一意のIPアドレスが割り当てられます。Podの各コンテナはIPアドレスとネットワークポートを含む、ネットワーク名前空間を共有します。*Podの中の* コンテナは`localhost`を使ってお互いに通信できます。Podのコンテナが *Podの外の* エンティティと通信する場合、どのように共有ネットワークリソースを使うのか (ポートなど) を調整しておかなければなりません。

#### ストレージ {#storage}

Podは共有ストレージ *ボリューム* のセットを指定できます。Pod内のすべてのコンテナは共有ボリュームにアクセスできます。ボリュームはPod内の永続データが、コンテナの再起動が必要な場合でも残るようにもします。KubernetesがどのようにPod内での共有ストレージを扱うのかは[ボリューム](/ja/docs/concepts/storage/volumes/)を参照してください。

## Podを利用する {#working-with-pods}

シングルトンPodであっても、個々のPodを直接作成することはめったにありません。その理由は、Podが比較的短命で、使い捨てのエンティティとして設計されているからです。Podが (ユーザによって直接またはコントローラによって間接的に) 作られると、クラスタ内のノードで実行するようスケジュールされます。Podは、プロセスが終了するか、Podオブジェクトが削除されるか、リソース不足のために *退去* させられるか、ノードに障害が起こるまでそのノードに残りつづけます。

{{< note >}}
**メモ:** Pod内のコンテナの再起動とPodの再起動を混同してはいけません。Pod自身が実行するのではなく、コンテナを実行し、削除されるまで持続する環境なのです。
{{< /note >}}

Podは自身で自己回復をしません。Podが障害のあるノードにスケジュールされたり、スケジューリング自体が失敗すると、Podは削除されます。Kubernetesは使い捨てのPodインスタンスを管理する、*コントローラ* と呼ばれる高レベルの抽象概念を使います。したがって、Podを直接使うことはできますが、KubernetesではPodを管理するためにコントローラを使うのがずっと一般的です。KubernetesがどのようにPodのスケーリングと回復を行うためにコントローラを使っているのかは[Podとコントローラ](#pods-and-controllers)を参照してください。

### Podとコントローラ {#pods-and-controllers}

コントローラは複数のPodを作成、管理できます。これは、複製とロールアウトを扱い、クラスタレベルでの自己回復能力を提供します。例えば、ノードに障害が起こると、コントローラはそこにあったPodと同じPodを異なるノードにスケジューリングすることで、自動的に置き換えます。

1つまたは複数のPodで構成されるコントローラの例には次のようなものがあります。

* [Deployment](/docs/concepts/workloads/controllers/deployment/)
* [StatefulSet](/ja/docs/concepts/workloads/controllers/statefulset/)
* [DaemonSet](/ja/docs/concepts/workloads/controllers/daemonset/)

通常、コントローラは作成するPodを指定するためにPodテンプレートを使います。

## Podテンプレート {#pod-template}

Podテンプレートは、[Replicationコントローラ](/docs/concepts/workloads/controllers/replicationcontroller/)や[Job](/docs/concepts/workloads/controllers/jobs-run-to-completion/)、[DaemonSet](/ja/docs/concepts/workloads/controllers/daemonset/)といった他のオブジェクトに含まれるPodのスペックです。コントローラは実際のPodを作るためにPodテンプレートを使います。下の例は、メッセージを表示するコンテナを含むPodの単純なマニフェストです。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

Podテンプレートは、すべてのレプリカの現在必要とする状態を指定するものというよりは、むしろ、クッキー型のようなものです。一度クッキーの型をとると、そのクッキーはクッキー型との関係がなくなります。「量子絡み合い」はありません。テンプレートへの後の変更や新しいテンプレートへの変更は、すでに作成されたPodに直接影響を与えることはありません。同様にReplicationコントローラによって作成されたPodは後で直接更新されるかもしれません。これはPodとの意図的な対照で、Podに属するすべてのコンテナの現在必要とする状態を指定します。このアプローチはシステムを徹底的にシンプルにし、柔軟性を増加させます。

{{% /capture %}}

{{% capture whatsnext %}}
* Podのふるまいの詳細については以下を参照してください
  * [Pod Termination](/docs/concepts/workloads/pods/pod/#termination-of-pods)
  * 他のPodのトピック
{{% /capture %}}
