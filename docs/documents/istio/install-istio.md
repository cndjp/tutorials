Istioのインストール
===================
このチュートリアルでは、Kubernetesクラスターにサービスメッシュ"Istio"をインストールします。


前提条件
--------
- Kubernetesクラスター 1.7.x+
- 上記Kubernetesクラスターに接続可能なkubectl

!!! Note
    本チュートリアルで、[ローカルPCにKubernetesクラスターを構成する手順](../../index.md#appendix)もご案内していますので、適宜ご利用ください。

!!! Warning
    Windows PCで、本チュートリアルの[Vagrant CoreOS Cluster](../create-local-cluster/vagrant-coreos-cluster.md)を利用している場合、インストールスクリプトの不具合と思われる問題により、このハンズオンを実施することができません。

    該当する場合、[minikube](../create-local-cluster/minikube.md)などの代替手段をご利用ください。

以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。


1. 準備作業: Kubernetesクラスターの設定を変更する
-------------------------------------------------
まず、KubernetesクラスターにIstioをインストールするために、クラスターの設定を変更します。

以下は、本チュートリアルの[ローカルPCにKubernetesクラスターを構成する手順](../../index.md#appendix)を利用した場合の方法を記します。
それ以外の環境を利用する場合については、[公式のドキュメントのPrerequisites](https://istio.io/docs/setup/kubernetes/quick-start.html#prerequisites)を参照ください。

### 1.1. [Vagrant CoreOS Cluster](../create-local-cluster/vagrant-coreos-cluster.md)の場合（Mac/Linuxのみ）

クラスターが起動中であれば、始めに停止します。

Vagrant CoreOS Clusterのインストールスクリプトを``clone``してできたディレクトリを、カレントディレクトリにします。以下、ユーザーホームディレクトリで``clone``している場合のコマンド例です。

    cd ~/kubernetes-vagrant-coreos-cluster

以下のコマンドで、クラスターを停止します。

    vagrant halt

クラスターの停止が完了したら、AUTHORIZATION\_MODE=RBACオプションを指定して、RBACを有効にします。

    AUTHORIZATION_MODE=RBAC vagrant up

通常どおりクラスターが起動するのを待ってから、[2. Istioのインストール](#2-istio)に進んでください。

### 1.2. [minikube](../create-local-cluster/minikube.md)の場合
クラスターが起動中であれば、始めに停止します。minikubeクラスターを停止するには、以下のコマンドを実行します。

    minikube stop

クラスターの停止後、以下のようにオプションを追加してクラスターを起動します。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    minikube start \
        --extra-config=controller-manager.ClusterSigningCertFile="/var/lib/localkube/certs/ca.crt" \
        --extra-config=controller-manager.ClusterSigningKeyFile="/var/lib/localkube/certs/ca.key" \
        --extra-config=apiserver.Admission.PluginNames=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
        --extra-config=apiserver.Authorization.Mode=RBAC \
        --bootstrapper=localkube \
        --kubernetes-version=v1.9.0

:fa-windows: __Windows__

    minikube start `
        --extra-config=controller-manager.ClusterSigningCertFile="/var/lib/localkube/certs/ca.crt" `
        --extra-config=controller-manager.ClusterSigningKeyFile="/var/lib/localkube/certs/ca.key" `
        --extra-config=apiserver.Admission.PluginNames=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota `
        --extra-config=apiserver.Authorization.Mode=RBAC `
        --bootstrapper=localkube `
        --kubernetes-version=v1.9.0

通常どおりクラスターが起動するのを待ってから、[2. Istioのインストール](#2-istio)に進んでください。


2. Istioのインストール
----------------------
ここでは、KubernetesにIstioをインストールします。

### 2.1. Istioのダウンロードとistioctlのセットアップ
Isitoは、Kubernetesにデプロイするためのmanifestやコマンドラインツール(istioctl)、サンプルアプリ等の一式をzipアーカイブにまとめたものとして配布されています。
このチュートリアルの作成時点(2018/04/20)では、0.7.1が最新バージョンです。

それでは、Istioをダウンロードします。Mac/Linuxの場合は、最新バージョンを取得とアーカイブの展開を自動的に実行してくれるスクリプトツールを利用することができます。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    curl -L https://git.io/getLatestIstio | sh -

Windowsの場合は、zipアーカイブのダウンロードや展開などの処理を個別のコマンドとして実行します。

:fa-windows: __Windows__

```bat
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; `
Invoke-WebRequest -Uri https://github.com/istio/istio/releases/download/0.7.1/istio-0.7.1-win.zip -OutFile istio_0.7.1_win.zip; `
Expand-Archive -Path .\istio_0.7.1_win.zip; `
mv .\istio_0.7.1_win\istio-0.7.1\ .\; rmdir .\istio_0.7.1_win; rm .\istio_0.7.1_win.zip
```

Istioのコマンドラインツール(istioctl)は、ダウンロードしてできたディレクトリ配下の``istio-0.7.1/bin``に配置されています。各プラットフォームに合わせて、istioctlをPATHに追加してください。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    export PATH=$PATH:$PWD/istio-0.7.1/bin

:fa-windows: __Windows__

    Set-Item Env:Path "$Env:Path;$(Convert-Path .)\istio-0.7.1\bin"

istioctlが使えることを確認するには、``istioctl version``を実行してistioctlのバージョン情報を表示します。

    istioctl version

以下のような結果となることを確認してください。

    Version: 0.7.1
    GitRevision: 62110d4f0373a7613e57b8a4d559ded9cb6a1cc8
    User: root@c5207293dc14
    Hub: docker.io/istio
    GolangVersion: go1.9
    BuildStatus: Clean

これで、Istioのダウンロードとistioctlのセットアップは完了です。

### 2.2. IstioをKubernetesクラスターにインストールする
IstioをKubernetesクラスターにインストールします。必要なものは全てKubernetesのmanifestファイルとして記述されているので、これをデプロイするだけでインストールが完了します。

    kubectl apply -f ./istio-0.7.1/install/kubernetes/istio.yaml

istio-systemというNamespace配下に必要な構成要素が配備されます。ServiceとPodにどのようなオブジェクトが配備されるか、確認してみます。

まず、Serviceオブジェクトです。

    kubectl get services -n istio-system

結果は以下のようになります。

    NAME            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                            AGE
    istio-ingress   LoadBalancer   10.100.151.26    <pending>     80:31826/TCP,443:31716/TCP                                         1m
    istio-mixer     ClusterIP      10.100.149.13    <none>        9091/TCP,15004/TCP,9093/TCP,9094/TCP,9102/TCP,9125/UDP,42422/TCP   1m
    istio-pilot     ClusterIP      10.100.118.141   <none>        15003/TCP,8080/TCP,9093/TCP,443/TCP                                1m

次にPodです。

    kubectl get pods -n istio-system

結果は以下のようになります。

    NAME                             READY     STATUS    RESTARTS   AGE
    istio-ca-418645489-gg8sh         1/1       Running   0          2m
    istio-ingress-1688831668-t24gh   1/1       Running   0          2m
    istio-mixer-3303323913-s8c2l     3/3       Running   0          2m
    istio-pilot-3625647026-0rv6z     2/2       Running   0          2m

READYの列は起動が確認されたPod内のContainerの数を表します。上のように全てのContainerがREADYになればインストール完了です（およそ1-2分ほどかかります）。

---
以上で、KubernetesへのIstioのインストールは完了です。
