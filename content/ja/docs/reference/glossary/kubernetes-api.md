---
title: Kubernetes API
id: kubernetes-api
date: 2018-04-12
full_link: /docs/concepts/overview/kubernetes-api/
short_description: >
  RESTfulインタフェースを通じてKubernetesの機能を提供し、クラスタの状態を格納するアプリケーション

aka: 
tags:
- fundamental
- architecture
---
 RESTfulインタフェースを通じてKubernetesの機能を提供し、クラスタの状態を格納するアプリケーション。

<!--more--> 

Kubernetesリソースと「意図の記録」はすべてAPIオブジェクトとして格納され、APIへのRESTful呼び出しによって変更されます。APIは構成を宣言的な方法で管理できるようにします。ユーザはKubernetes APIと直接、または`kubectl`のようなツールを通じてやりとりできます。コアKubernetes APIは柔軟で、カスタムリソースをサポートするために拡張することもできます。
