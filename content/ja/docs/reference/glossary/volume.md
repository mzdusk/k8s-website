---
title: Volume
id: volume
date: 2018-04-12
full_link: /docs/concepts/storage/volumes/
short_description: >
  Pod内のコンテナからアクセス可能な、データを含むディレクトリ

aka: 
tags:
- core-object
- fundamental
---
 {{< glossary_tooltip text="pod" term_id="pod" >}}内のコンテナからアクセス可能な、データを含むディレクトリ

<!--more--> 

Kubernetes VolumeはPodと同じ長さの生存期間を持ちます。したがって、{{< glossary_tooltip text="Pod" term_id="pod" >}}内で稼動している{{< glossary_tooltip text="コンテナ" term_id="container" >}}よりも長生きで、データは{{< glossary_tooltip text="コンテナ" term_id="container" >}}が再起動されても保持されます。
