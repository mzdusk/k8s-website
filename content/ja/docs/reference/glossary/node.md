---
title: ノード
id: node
date: 2018-10-15
full_link: /docs/concepts/architecture/nodes/
short_description: >
  ノードはKubernetesのワーカマシンです。

aka:
tags:
- fundamental
---
 ノードはKubernetesのワーカマシンです。

<!--more-->

クラスタによって、ワーカマシンはVMでも物理マシンでも構います。{{< glossary_tooltip text="Pod" term_id="pod" >}}を実行するのに必要な{{< glossary_tooltip text="サービス" term_id="service" >}}を持ち、マスタコンポーネントによって管理されます。ノードの{{< glossary_tooltip text="サービス" term_id="service" >}}にはDockerやkubelet、kube-proxyがあります。
