---
title: Pod
id: pod
date: 2018-04-12
full_link: /ja/docs/concepts/workloads/pods/pod-overview/
short_description: >
  最も小さくてシンプルなKubernetesオブジェクト。Podはクラスタ上で稼動するコンテナのセットを表現します。

aka: 
tags:
- core-object
- fundamental
---
 最も小さくてシンプルなKubernetesオブジェクト。Podはクラスタ上で稼動する{{< glossary_tooltip text="コンテナ" term_id="container" >}}のセットを表現します。

<!--more--> 

Podは通常、単一のコンテナを実行するためにセットアップされます。ロギングのような補助機能を追加する、運用上のサイドカーコンテナを実行することもできます。Podは一般的には{{< glossary_tooltip term_id="deployment" >}}によって管理されます。

