---
title: DaemonSet
id: daemonset
date: 2018-10-15
full_link: /ja/docs/concepts/workloads/controllers/daemonset
short_description: >
  Podの1つ複製がクラスタ内のノードのセットで確実に実行されるようにします。

aka: 
tags:
- fundamental
- core-object
- workload
---
 {{< glossary_tooltip text="Pod" term_id="pod" >}}の1つ複製が{{< glossary_tooltip text="クラスタ" term_id="cluster" >}}内のノードのセットで確実に実行されるようにします。

<!--more--> 

通常すべての{{< glossary_tooltip term_id="node" >}}で実行しなければならない、ログ収集やモニタリングエージェントのようなシステムデーモンのデプロイに使われます。
