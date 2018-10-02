---
title: Secret
id: secret
date: 2018-04-12
full_link: /docs/concepts/configuration/secret/
short_description: >
  パスワードやOAuthトークン、ssh鍵のような機密情報を格納します。

aka: 
tags:
- core-object
- security
---
 パスワードやOAuthトークン、ssh鍵のような機密情報を格納します。

<!--more--> 

[暗号化](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#ensure-all-secrets-are-encrypted)を含む、機密情報の使用方法を管理し、偶発的な公開のリスクを軽減します。{{< glossary_tooltip text="Pod" term_id="pod" >}}はVolumeにマウントしたファイルや、kubeletがPodに対してイメージをPullすることでSecretを参照します。Secretは機密データに最適で、[ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)は非機密データに最適です。
