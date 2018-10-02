---
title: Persistent Volume
id: persistent-volume
date: 2018-04-12
full_link: /docs/concepts/storage/persistent-volumes/
short_description: >
  クラスタのストレージの一部を表現するAPIオブジェクト。個々のPodのライフサイクルを越えて永続するプラガブルリソースとして一般的に利用可能。

aka: 
tags:
- core-object
- storage
---
 クラスタのストレージの一部を表現するAPIオブジェクト。個々の{{< glossary_tooltip text="Pod" term_id="pod" >}}のライフサイクルを越えて永続するプラガブルリソースとして一般的に利用可能。

<!--more--> 

PersistentVolume (PV) はどのようなストレージがどのように提供されているのかの詳細を抽象化するAPIを提供します。PVはストレージを事前に作成するシナリオでは直接使われます (静的プロビジョニング)。オンデマンドストレージが必要なシナリオでは (動的プロビジョニング)、かわりにPersistentVolumeClaim (PVC) を使います。
