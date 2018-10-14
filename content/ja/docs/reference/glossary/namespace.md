---
title: 名前空間
id: namespace
date: 2018-10-15
full_link: /docs/concepts/overview/working-with-objects/namespaces
short_description: >
  同じ物理クラスタ上で複数の仮想的なクラスタをサポートするためにKubernetesによって使われる抽象概念

aka:
tags:
- fundamental
---
 同じ物理{{< glossary_tooltip text="クラスタ" term_id="cluster" >}}上で複数の仮想的なクラスタをサポートするためにKubernetesによって使われる抽象概念

<!--more-->

名前空間はクラスタでオブジェクトを組織化し、クラスタリソースを分離する方法を提供するために使われます。リソースの名前は名前空間内で一意である必要がありますが、名前空間外ではその限りではありません。
