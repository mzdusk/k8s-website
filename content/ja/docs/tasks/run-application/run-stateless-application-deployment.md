---
title: Deploymentを使ったステートレスアプリケーションの実行
min-kubernetes-server-version: v1.9
content_template: templates/tutorial
weight: 10
---

{{% capture overview %}}

このページではKubernetes Deploymentオブジェクトを使ったアプリケーションの実行方法を示します。

{{% /capture %}}

{{% capture objectives %}}

* nginx Deploymentを作成する
* Deploymentの情報を列挙するためにkubectlを使う
* Deploymentを更新する

{{% /capture %}}

{{% capture prerequisites %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

{{% /capture %}}

{{% capture lessoncontent %}}

## nginx Deploymentの作成と探索 {#creating-and-exploring-an-nginx-deployment}

Kubernetes Deploymentオブジェクトを作成することでアプリケーションを実行でき、DeploymentはYAMLファイルで記述できます。例えば、次のYAMLファイルはnginx:1.7.9 Dockerイメージを実行するDeploymentを記述します。

{{< codenew file="application/deployment.yaml" >}}

1. このYAMLファイルに基づくDeploymentを作成する

        kubectl apply -f https://k8s.io/examples/application/deployment.yaml

2. Deploymentの情報を表示する

        kubectl describe deployment nginx-deployment

    出力は次にようになります

        user@computer:~/website$ kubectl describe deployment nginx-deployment
        Name:     nginx-deployment
        Namespace:    default
        CreationTimestamp:  Tue, 30 Aug 2016 18:11:37 -0700
        Labels:     app=nginx
        Annotations:    deployment.kubernetes.io/revision=1
        Selector:   app=nginx
        Replicas:   2 desired | 2 updated | 2 total | 2 available | 0 unavailable
        StrategyType:   RollingUpdate
        MinReadySeconds:  0
        RollingUpdateStrategy:  1 max unavailable, 1 max surge
        Pod Template:
          Labels:       app=nginx
          Containers:
           nginx:
            Image:              nginx:1.7.9
            Port:               80/TCP
            Environment:        <none>
            Mounts:             <none>
          Volumes:              <none>
        Conditions:
          Type          Status  Reason
          ----          ------  ------
          Available     True    MinimumReplicasAvailable
          Progressing   True    NewReplicaSetAvailable
        OldReplicaSets:   <none>
        NewReplicaSet:    nginx-deployment-1771418926 (2/2 replicas created)
        No events.

3. Deploymentによって作成されたPodを列挙する

        kubectl get pods -l app=nginx

    出力は次にようになります

        NAME                                READY     STATUS    RESTARTS   AGE
        nginx-deployment-1771418926-7o5ns   1/1       Running   0          16h
        nginx-deployment-1771418926-r18az   1/1       Running   0          16h

4. Podの情報を表示する

         kubectl describe pod <pod-name>

    `<pod-name>`はPodの名前です

## Deploymentの更新 {#updating-the-deployment}

新しいYAMLファイルを適用することでDeploymentを更新することができます。このYAMLファイルはDeploymentにnginx 1.8を使うよう指示します。

{{< codenew file="application/deployment-update.yaml" >}}

1. この新しいYAMLファイルを適用する

        kubectl apply -f https://k8s.io/examples/application/deployment-update.yaml

2. Deploymentが新しい名前のPodを作成し、古いPodを削除するのを監視する

        kubectl get pods -l app=nginx

## レプリカ数の増加によるアプリケーションのスケーリング {#scaling-the-application-by-increasing-the-replica-count}

新しいYAMLファイルを適用することで、DeploymentのPodの数を増やすことができます。このYAMLファイルでは`replicas`を4に設定し、Deploymentに4つのPodを持つよう指示します。

{{< codenew file="application/deployment-scale.yaml" >}}

1. 新しいYAMLファイルを適用する

        kubectl apply -f https://k8s.io/examples/application/deployment-scale.yaml

2. Deploymentが4つのPodを持つことを確認する

        kubectl get pods -l app=nginx
    
    出力は次のようになります

        NAME                               READY     STATUS    RESTARTS   AGE
        nginx-deployment-148880595-4zdqq   1/1       Running   0          25s
        nginx-deployment-148880595-6zgi1   1/1       Running   0          25s
        nginx-deployment-148880595-fxcez   1/1       Running   0          2m
        nginx-deployment-148880595-rwovn   1/1       Running   0          2m

## Deploymentの削除 {#deleting-a-deployment}

名前を使ってDeploymentを削除します。

    kubectl delete deployment nginx-deployment

## ReplicationController -- 古い方法 {#replicationcontrollers-the-old-way}

複製されたアプリケーションを作成する推奨の方法はDeploymentを使うことですが、同様にReplicaSetを使うことができます。DeploymentとReplicaSetがKubernetesに追加される前は、複製されたアプリケーションを構成するために[ReplicationController](/docs/concepts/workloads/controllers/replicationcontroller/)を使っていました。

{{% /capture %}}

{{% capture whatsnext %}}

* [Deploymentオブジェクト](/docs/concepts/workloads/controllers/deployment/)についてさらに学びます

{{% /capture %}}
