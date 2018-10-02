---
title: ConfigMap
id: configmap
date: 2018-04-12
full_link: /docs/tasks/configure-pod-container/configure-pod-configmap/
short_description: >
  キーバリューペアで非機密情報を格納するために使うAPIオブジェクト。環境変数やコマンドライン引数、Volumeの構成ファイルとして利用可能。

aka: 
tags:
- core-object
---
 キーバリューペアで非機密情報を格納するために使うAPIオブジェクト。環境変数やコマンドライン引数、{{< glossary_tooltip text="Volume" term_id="volume" >}}の構成ファイルとして利用可能。

<!--more--> 

環境特有の構成から{{< glossary_tooltip text="コンテナイメージ" term_id="container" >}}を分離できるので、アプリケーションが容易にポータブルになります。機密データを格納する場合は[Secret](https://kubernetes.io/docs/concepts/configuration/secret/)を使ってください。
