---
title: 汚染と耐性
content_template: templates/concept
weight: 40
---

{{< toc >}}

{{% capture overview %}}

ノードアフィニティは、[ここ](/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature)で述べたように、ノードの集合にPodを *引き付ける* *Podの* プロパティです。汚染は逆で、これらは *ノード* にPodを *寄せつけない* ようにできます。

汚染と耐性は確実にPodを不適切なノードにスケジュールされないようにするために一緒に動作します。1つ以上の汚染がノードに適用され、これはそのノードが汚染に耐性のないPodを受け付けるべきでないことを示します。耐性はPodに適用され、マッチする汚染のノードにPodがスケジュールされてもよいことを示します。

{{% /capture %}}

{{% capture body %}}

## 概念

汚染は[kubectl taint](/docs/reference/generated/kubectl/kubectl-commands#taint)を使ってノードに付加できます。例えば、

```shell
kubectl taint nodes node1 key=value:NoSchedule
```

で`node1`に汚染を配置する。この汚染は`key`というキーと`value`という値と`NoSchedule`という汚染効果を持つ。これはマッチする耐性が無ければ、`node1`にPodをスケジュールできないということを意味する。

上で付加した汚染を除去するためには次を実行する。

```shell
kubectl taint nodes node1 key:NoSchedule-
```

Podに対する耐性はPodSpecに指定する。以下の耐性は両方とも上の`kubectl taint`で作成した汚染にマッチするので、どちらかの耐性を持つPodは`node1`にスケジュールされることができる。

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

キーと効果が同じで以下を満たせば、耐性は汚染にマッチする。

* `operator`が`Exists` (`value`を指定しないケース) または
* `operator`が`Equal`で`value`も同じ

`Operator`は指定しなければデフォルトの`Equal`となる。

{{< note >}}

**メモ:** 2つの特別なケースがあります

* `key`のない`Exists`演算子はすべてのキー、値、効果にマッチし、これはすべてを許容することを意味する

```yaml
tolerations:
- operator: "Exists"
```

* 空の`effect`は`key`というキーを持つすべての効果にマッチする

```yaml
tolerations:
- key: "key"
  operator: "Exists"
```
{{< /note >}}

上の例では`NoSchedule`の`effect`を使いました。ほかにも、`PreferNoSchedule`の`effect`も使えます。これは`NoSchedule`の「好み」もしくは「ソフト」バージョンです。システムはそのノードへの汚染に耐性のないPodの配置を避けようとするが、必須ではない。3種類目の効果は`NoExecute`で、これは後ほど言及する。

同じノードに複数の汚染と、同じPodに複数の耐性を持たせることができます。Kubernetesが複数の汚染と耐性を処理する方法はフィルタと似ています。ノードの汚染すべてから始め、マッチする耐性を持つPodを無視します。残りの無視されなかった汚染はそのPodに対して指示された効果をもたらします。特に、

* `NoSchedule`効果のある無視できない汚染が1つでもあれば、KubernetesはそのノードにPodをスケジュールしません
* `NoSchedule`効果のある無視できない汚染はないけれど、`PreferNoSchedule`効果のある無視できない汚染が1つでもあれば、KubernetesはそのノードにPodを配置しないよう *試み* ます
* `NoExecute`効果のある無視できない汚染が1つでもあれば、(すでに実行中なら) Podはノードから退去させられ、(まだ実行していなければ) そのノードにはスケジュールされません

例えば、次のようにノードを汚染するとし、

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

2つの耐性を持つPodがあるとします。

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

この場合、3つ目の汚染にマッチする耐性がないため、Podはこのノードにスケジュールされません。しかし、汚染が追加された時、すでに実行中であれば、実行し続けることができます。なぜなら、3つ目の汚染は、そのPodに耐性のない3つのうちの1つにすぎないためです。

通常、`NoExecute`効果を持つ汚染がノードに追加されると、その汚染に耐性のないPodはすぐに退去させられ、耐性のあるPodは退去させられません。しかし、`NoExecute`効果のある耐性は、汚染が追加されてからどれくらいの時間ノードに結合しつづけられるかを指示する、オプションの`tolerationSeconds`フィールドを指定できます。例えば、

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

は、このPodが起動していて、マッチする汚染がノードに追加されると、3600秒間ノードに結合しつづけ、その後退去させられます。それまでに汚染が除去されれば、Podは退去させられません。

## ユースケース例

汚染と耐性はPodをノードから *遠ざけ* たり、実行すべきでないPodを退去させたりするための柔軟な方法です。いくつかのユースケースを挙げます。

* **専用のノード**: ノードを特定のユーザに排他的に使わせたい場合、それらのノードに汚染を追加でき (`kubectl taint nodes nodename dedicated=groupName:NoSchedule`)、それらのPodには対応する耐性を追加します (これはカスタム[アドミッションコントローラ](/ja/docs/reference/access-authn-authz/admission-controllers/)を書くことで最も簡単に実現できます)。耐性のあるPodは汚染された(専用の)ノードを他のノードと同じように使うことができます。もし、ノードを専用にし、 *かつ* 専用ノード *のみ* を使わせたいのであれば、ラベルを汚染と同じノードに追加し (例えば`dedicated=groupName`)、アドミッションコントローラは`dedicated=groupName`とラベル付けされたノードにのみスケジュールできるようノードアフィニティを追加すべきです。

* **特別なハードウェアのノード**: 少数のノードが特別なハードウェア (GPUなど) をもつクラスタでは、その特別なハードウェアが必要ないPodを遠ざけ、後から来る特別なハードウェアを必要とするPodのために空けておくのが望ましいでしょう。これは特別なハードウェアを持つノードの汚染 (例えば `kubectl taint nodes nodename special=true:NoSchedule` または `kubectl taint nodes nodename special=true:PreferNoSchedule`) と特別なハードウェアを使うPodへの対応する耐性の追加で実現でき、これはカスタム[アドミッションコントローラ](/ja/docs/reference/access-authn-authz/admission-controllers/)を使って耐性を適用するのがおそらく最も簡単でしょう。例えば、特別なハードウェアを表現するには[拡張リソース](/ja/docs/concepts/configuration/manage-compute-resources-container/#extended-resources)を使い、拡張リソース名で特別なハードウェアのノードを汚染し、[ExtendedResourceToleration](/ja/docs/reference/access-authn-authz/admission-controllers/#extendedresourcetoleration)アドミッションコントローラを実行することが推奨されます。これで、ノードが汚染されるので、耐性のないPodはそこにスケジュールされません。しかし、拡張リソースを必要とするPodが追加されると、`ExtendedResourceToleration`アドミッションコントローラは、特別なハードウェアのノードにそのPodがスケジュールされるよう適切な耐性を自動的に付与します。これで特別なハードウェアのノードはそのハードウェアを要求するPod専用になり、手動でPodに耐性を付与する必要もありません。

* **汚染ベースの退去 (アルファ機能)**: ノード問題があった時のPodごとの構成による退去のふるまいで、次のセクションで述べます。

## 汚染ベースの退去

以前に`NoExecute`汚染効果について言及しました。これはすでにノードで実行しているPodに影響します。

* 汚染に耐性のないPodは即座に退去させられる
* 耐性スペックに`tolerationSeconds`が指定されていない汚染に耐性のあるPodは永遠に結合したままになる
* `tolerationSeconds`が指定された汚染に耐性のあるPodは指定された時間だけ結合したままになる

加えて、Kubernetes 1.6ではノード問題の表現に対するアルファサポートが導入されました。言い換えると、ノードコントローラは特定の条件が真になると自動的にノードを汚染します。以下の汚染が組込まれています。

* `node.kubernetes.io/not-ready`: ノードは準備ができていません。これはNodeCondition `Ready`が"`False`"になっている状態です。
* `node.kubernetes.io/unreachable`: ノードコントローラからノードに到達不能です。これはNodeCondition `Ready`が"`Unknown`"になっている状態です。
* `node.kubernetes.io/out-of-disk`: ノードがディスク不足になっています。
* `node.kubernetes.io/memory-pressure`: ノードがMemory Pressureです。
* `node.kubernetes.io/disk-pressure`: ノードがDisk Pressureです。
* `node.kubernetes.io/network-unavailable`: ノードのネットワークが利用できません。
* `node.kubernetes.io/unschedulable`: ノードがスケジュール不可です。
* `node.cloudprovider.kubernetes.io/uninitialized`: kubeletが"外部の"クラウドプロバイダで開始されると、この汚染が使用不可としてノードに設定されます。cloud-controller-managerからのコントローラがこのノードを初期化したあと、kubeletはこの汚染を除去します。

`TaintBasedEvictions`アルファ機能が有効であれば (これは、`--feature-gates=FooBar=true,TaintBasedEvictions=true`のようにKubernetesコントローラマネージャの`--feature-gates`に`TaintBasedEvictions=true`を含めることで実現できます)、汚染はNodeController (もしくはkubelet) によって自動的に付加され、Ready NodeConditionに基づくノードからPodを退去させる通常のロジックは無効にされます。

{{< note >}}
**メモ:** ノード問題のためのPod退去における既存の[速度制限](/ja/docs/concepts/architecture/nodes/)のふるまいを維持するために、システムは、速度制限された方法で汚染を追加します。これは、マスタがノードから分断されるような状況による大規模なPodの退去を防ぎます。
{{< /note >}}

このアルファ機能は、`tolarationSeconds`と併用することで、これらの問題を持つノードにどれくらい結合しつづけるのかをPodに指定できます。

例えば、多くのローカル状態を持つアプリケーションはネットワークの分断があっても、分断が解消してPodの退去が避けられることを期待して、長い間ノードに結合しつづけておきたい場合があります。このような場合に使われるPodの耐性は次のようになります。

```yaml
tolerations:
- key: "node.alpha.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

Kubernetesは、ユーザによって提供されたPodスペックにすでに`node.kubernetes.io/net-ready`の耐性がなければ、`tolerationSeconds=300`である`node.kubernetes.io/not-ready`の耐性を自動的に付加することに注意してください。同様にユーザによって提供されたPodスペックにすでに`node.alpha.kubernetes.io/unreachable`の耐性がなければ、`tolerationSeconds=300`である`node.alpha.kubernetes.io/unreachable`の耐性を自動的に付加します。

これらの自動的に不可される耐性は、これらの問題が検出されてから5分間結合しつづけるというデフォルトのPodのふるまいを維持します。2つのデフォルト耐性が[DefaultTolerationSecondsアドミッションコントローラ](https://git.k8s.io/kubernetes/plugin/pkg/admission/defaulttolerationseconds)によって追加されます。

[DaemonSet](/ja/docs/concepts/workloads/controllers/daemonset/) Podは`tolerationSeconds`なしの以下の汚染に対する`NoExecute`耐性を持って作成されます。

  * `node.alpha.kubernetes.io.unreachable`
  * `node.kubernetes.io/not-ready`

## 条件によるノードの汚染 {#taint-nodes-by-condition}

バージョン1.12では、`TaintNodesByCondition`機能がベータに昇格したので、ノードライフサイクルコントローラはノード条件に応じて汚染を自動的に作成します。同様に、スケジューラはノード条件をチェックしません。かわりに汚染をチェックします。これはノード条件がノードにスケジュール済みのものに対して確実に影響を与えないようにします。ユーザは適切なPod耐性を追加することで、(ノード条件として表現される) いくつかのノード問題を無視することができます。`TaintNodesByCondition`は`NoSchedule`効果のあるノードのみを汚染することに注意してください。`NoExecute`効果は、アルファ機能でデフォルトでは無効になっている`TaintBasedEvivsion`によって制御されます。

Kubernetes 1.8から、DaemonSetコントローラは、DaemonSetが壊れるのを防ぐために、以下の`NoSchedule`耐性をすべてのデーモンに自動的に付加します。

  * `node.kubernetes.io/memory-pressure`
  * `node.kubernetes.io/disk-pressure`
  * `node.kubernetes.io/out-of-disk` (*critical podのみ*)
  * `node.kubernetes.io/unschedulable` (1.10以降)
  * `node.kubernetes.io/network-unavailable` (*host networkのみ*)

これらの耐性を付加することで後方互換性を保証します。任意の耐性をDaemonSetに付加することもできます。

{{% /capture %}}
