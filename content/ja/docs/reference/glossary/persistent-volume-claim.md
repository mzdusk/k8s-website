---
title: Persistent Volume Claim
id: persistent-volume-claim
date: 2018-04-12
full_link: /docs/concepts/storage/persistent-volumes/
short_description: >
  コンテナでVolumeとしてマウントできるようにPerisstentVolumeで定義されたストレージリソースを請求します。

aka: 
tags:
- core-object
- storage
---
 コンテナでVolumeとしてマウントできるようにPerisstentVolumeで定義されたストレージリソースを請求します。

<!--more--> 

ストレージの容量や、ストレージにアクセスする方法 (参照のみ、読み書きと実行可能かどうか) 、再請求の方法 (Retained、Recycled、Deleted) を指定します。ストレージ自身の詳細はPersistentVolumeスペックにあります。
