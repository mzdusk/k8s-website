---
title: StatefulSet
id: statefulset
date: 2018-04-12
full_link: /ja/docs/concepts/workloads/controllers/statefulset/
short_description: >
  DeploymentとPodの集合のスケーリングを管理し、 *これらのPodの順序と一意性についての保証を提供* します。

aka: 
tags:
- fundamental
- core-object
- workload
- storage
---
 Deploymentと{{< glossary_tooltip text="Pod" term_id="pod" >}}の集合のスケーリングを管理し、 *これらのPodの順序と一意性についての保証を提供* します。

<!--more--> 

{{< glossary_tooltip term_id="deployment" >}}のように、StatefulSetは同じコンテナスペックに基づくPodを管理します。Deploymentと違い、StatefulSetは各Podの継続的なIDを維持します。これらのPodは同じスペックから作られるが、互換性はありません。それぞれがあらゆる再スケジューリングを越えても維持される永続的なIDを持ちます。

StatefulSetは他のコントローラと同じパターンの元で操作します。StatefulSetオブジェクトに求める状態を定義し、StatefulSet *コントローラ* は現在の状態からそこに向かうために必要な更新を行います。
