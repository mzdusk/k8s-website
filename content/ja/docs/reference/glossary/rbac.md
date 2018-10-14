---
title: RBAC (Role-Based Access Control)
id: rbac
date: 2018-10-15
full_link: /ja/docs/reference/access-authn-authz/rbac/
short_description: >
  管理者がKubernetes APIを通じて動的にアクセスポリシを構成できるよう、認可判定を管理します。

aka:
tags:
- security
- fundamental
---
 管理者が{{< glossary_tooltip text="Kubernetes API" term_id="kubernetes-api" >}}を通じて動的にアクセスポリシを構成できるよう、認可判定を管理します。

<!--more-->

RBACは権限ルールを含む *ロール* と、ユーザの就業に定義した権限を与える *ロール結合* を使います。
