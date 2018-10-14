---
title: サービスアカウント
id: service-account
date: 2018-10-15
full_link: /docs/tasks/configure-pod-container/configure-service-account/
short_description: >
  Pod内で実行するプロセスのIDを提供します。

aka:
tags:
- fundamental
- core-object
---
 {{< glossary_tooltip text="Pod" term_id="pod" >}}内で実行するプロセスのIDを提供します。

<!--more-->

Pod内のプロセスがクラスタにアクセスする場合、`default`のような特定のサービスアカウントとしてAPIサーバに認証されます。Podを作成する時にサービスアカウントを指定しなければ、その{{< glossary_tooltip text="名前空間" term_id="namespace" >}}のデフォルトサービスアカウントに自動的に割り当てられます。
