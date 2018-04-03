最初のアプリケーションを動かしてみる
====================================
このチュートリアルでは、簡単なサンプルアプリケーションを使って、Kubernetes上でコンテナを動かすオペレーションを体験します。


前提条件
--------
- 任意のKubernetesクラスター
- 上記Kubernetesクラスターに接続可能なkubectl

!!! Note
    本チュートリアルで、ローカルPCにKubernetesクラスターを構成する手順もご案内していますので、適宜ご利用ください。


1. サンプルアプリケーションのデプロイと動作確認
-----------------------------------------------
ここでは、簡単なサンプルアプリケーション(bootcamp)をデプロイして、クラスターの動作確認をします。

### 1.1. bootcampのデプロイ
Kubernetesクラスターにサンプルアプリケーションをデプロイしてみます。以下のコマンドを実行します。

```sh
kubectl run kubernetes-bootcamp --image=docker.io/jocatalin/kubernetes-bootcamp:v1 --port=8080
```

上記コマンドにより、サンプルアプリケーションのデプロイを管理する"Deployment"オブジェクトが作成されています。"Deployment"オブジェクトの情報を確認するために、以下のコマンドを実行してみてください。

__Deploymentの一覧の取得__

    kubectl get deployments

__Deploymentの詳細情報の取得__

    kubectl describe deployments/kubernetes-bootcamp

!!!Note
    このアプリケーションは、[Kubernetesの公式のインタラクティブ・チュートリアル](https://kubernetesbootcamp.github.io/kubernetes-bootcamp/)で利用しているものと同じものです。

### 1.2. bootcampへのアクセス
アプリケーションにクラスターの外からアクセスするために、ここではkubectlの機能を使ってクラスター内に通じるプロキシを構成します。以下のコマンドを実行すると、localhostの8001ポートからクラスター内にアクセスできるようになります。

    kubectl proxy

まずは、Kubernetesのマスターノードで稼働しているapiserverに対してリクエストを送信し、プロキシが機能していることを確認します。

:fa-apple: __Mac__ / :fa-linux: __Linux__

```sh
curl http://localhost:8001/version
```

:fa-windows: __Windows__

```bat
Invoke-RestMethod -Uri "http://localhost:8001/version"
```

Kubernetesのバージョン情報が、以下のようなJSON形式で取得できます。

```json
{
  "major": "1",
  "minor": "7",
  "gitVersion": "v1.7.5",
  "gitCommit": "17d7182a7ccbb167074be7a87f0a68bd00d58d97",
  "gitTreeState": "clean",
  "buildDate": "2017-08-31T08:56:23Z",
  "goVersion": "go1.8.3",
  "compiler": "gc",
  "platform": "linux/amd
}
```

実際にアプリケーションにアクセスしてみます。以下のコマンドを順に実行してください。

:fa-apple: __Mac__ / :fa-linux: __Linux__

```sh
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```
```sh
curl http://localhost:8001/api/v1/proxy/namespaces/default/pods/$POD_NAME/
```

:fa-windows: __Windows__

```bat
$POD_NAME=kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{\"\n\"}}{{end}}'
```
```bat
Invoke-RestMethod -Uri "http://localhost:8001/api/v1/proxy/namespaces/default/pods/$POD_NAME/"
```

正しく動作していれば、以下のようなレスポンスが返却されます。

    Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-2457653786-vdgr2 | v=1

これで、bootcampのバージョン1が動作していることが確認できました。

!!!Note
    ``kubectl proxy``コマンドを使ったこの方法は、基本的にkubectlインストールされた環境でしか利用でませんので、アプリを一般公開するために使うことはできません。
    管理用のダッシュボードや、Grafanaのような監視用のGUIにアクセスするときに利用するのが、一般的な用途です。

    アプリケーションを一般に公開するには、後述するServiceオブジェクト等を利用します。


2. Podの状態の確認
------------------
ここでは、クラスター上で動作しているbootcampの動作状況を、Podの標準出力を表示するなどして確認してみます。

### 2.1. Podの標準出力の表示
Kubernetes上で動作するアプリケーションの動作状況を確認する上で、最もシンプルな方法は、Podの標準出力確認することです。Podの標準出力を表示するには、以下のコマンドを実行します。

    kubectl logs [Pod名]

前章で、環境変数POD_NAMEに、現在稼働しているbootcampのPod名を保存しましたので、以下のようにコマンドを呼び出すことができます。

    kubectl logs $POD_NAME

### 2.2. Podの環境変数の確認
Podに設定されている環境変数を確認するには、Pod内にアクセスして``env``コマンドを実行する必要があります。

まず、Pod内から任意のコマンドを実行するには``kubectl exec``コマンドを用います。

    kubectl exec [Pod名] [実行したいコマンド]

[実行したいコマンド]に``env``を当てはめて実行すると、指定したPod内でそれが呼び出され、環境変数を出力することができます。

    kubectl exec $POD_NAME env

``kubectl exec``を利用すると、Podのシェルを利用することもできます。

    kubectl exec -it $POD_NAME /bin/bash

### 2.3. 複数のPodの標準出力を一度にtailする
Podをスケールアウトすると、複数のPodの標準出力を同時に観察したい場面が出てきます。ここまでで紹介した方法では、個別にコマンドラインツールを立ち上げて、``kubectl logs``とする必要がありますが、[Stern](https://github.com/wercker/stern)を使うとこの作業が容易になります。Sternは複数のPodの標準出力を、ひとつのコンソール上でまとめてtailするツールです。

Sternをインストールするには、利用しているPCの環境に合わせて以下の手順を実施します。

:fa-apple: __MacOS X__

homebrewを使ってインストールします。

    brew install stern

:fa-windows: __Windows__ / :fa-linux: __Linux__

ビルド済みのバイナリを以下からダウンロードしてPATHを通します。

- [Sternのバイナリ](https://github.com/wercker/stern/releases)

インストールが完了したら、以下のコマンドを実行してsternが利用できることを確認します。

    stern -v

!!! Note
    sternはデフォルトでkubectlの設定情報を利用して動作しますので、kubectlが利用できていれば、追加の設定作業は必要ありません。

まずは、ひとつのPodの名前を指定してsternを実行してみます。以下のコマンドを実行して、``kubectl logs``のときと同様の内容が表示されることを確認してください。

    stern $POD_NAME

Podの名前にワイルドカードを指定すると、それにマッチする複数のPodの標準出力を、同時に出力することができます。

    stern kubernetes-bootcamp-*


3. Serviceを使ったアプリケーションの公開
----------------------------------------
Serviceは、クラスター内外にPodを公開するために利用するKubernetesオブジェクトです。ここでは、Serviceを利用して、bootcampアプリをクラスター外に公開してみます。

### 3.1. Serviceの作成
Serviceを作成するには、``kubectl expose``を利用します。以下のコマンドを実行してください。

    kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080

上記のコマンドでは、bootcampをデプロイするときに作成したDeploymentに対して、Serviceを構成しています。このDeploymentにスケールアウトを指示すると、自動的にDeploymentが管理する複数のPodがルーティング対象となります。

ServiceのタイプにはNodePortを指定しています。これはクラスターを構成する全てのNodeの所定のPort番号から、当該Serviceにアクセスできるようにする設定です。

--port 8080のオプションは、このサービスに対して8080番ポートでアクセスできることを意味しています。

以下のコマンドで、指定したServiceが構成されていることを確認します。

__Serviceの一覧__

    kubectl get services

__Serviceの詳細情報__

    kubectl describe services/kubernetes-bootcamp

!!! Note
    Serviceは、特別な場合を除いて、クラスター内から参照可能なIPアドレス(ClusterIP)とポート番号を持ちます。NodePortタイプのServiceを構成すると、Nodeの所定のポート番号に対するアクセスを、ServiceのClusterIPとポート番号に転送するように動作します。

    Service作成時に指定したポート番号は、ServiceのClusterIPに紐づくポート番号であり、クラスター外に公開されているNodeのポート番号とは別のものだということに注意してください。

### 3.2. Serviceを介したアプリケーションへのアクセス
NodePortモードでServiceを構成していますので、Nodeの所定のポート番号から、アプリケーションにアクセスすることができます。

以下、この後の手順を簡単にするため、必要な値を環境変数に入れておきます。まず、Nodeの公開されているポート番号を、環境変数NODE\_PORTに保存します。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')

:fa-windows: __Windows__

    $NODE_PORT=kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}'

続いて、Nodeのホスト名を、環境変数NODE01に保存します。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    export NODE01=172.17.8.102
    export NODE02=172.17.8.103

:fa-windows: __Windows__

    $NODE01=echo '172.17.8.102'
    $NODE02=echo '172.17.8.103'

NodeのIPアドレス/ポート番号に対してHTTPリクエストを送信すると、1.で``kube proxy``を利用したときと同様の、bootcampの応答が返却されます。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    curl http://${NODE01}:${NODE_PORT}/

:fa-windows: __Windows__

    Invoke-RestMethod -Uri "http://${NODE01}:${NODE_PORT}/"

マルチノードのクラスターを構成している場合は、アクセス先のNodeを変更しても同じように応答が返却されることも確認してみてください。


4. アプリケーションのスケールアウト
-----------------------------------
この時点では、bootcampはひとつのPodで稼働している状態です。ここでは、Deploymentに対してレプリカの数を指定することによって、Podをスケールアウトしてみます。

### 4.1. Podの追加
Deploymentに対してレプリカの数を指定することによって、そのDeploymentが管理するPodの数を増減することができます。

レプリカの数を変更するには、``kubectl scale``コマンドを使用します。以下のように実行することで、bootcampのPodを管理するDeploymentに対して、レプリカ数を4にするよう指示します。

    kubectl scale deployments/kubernetes-bootcamp --replicas=4

Podの一覧を表示してみます。

    kubectl get pods

すると、4つのPodが構成されていることがわかります。

    NAME                                   READY     STATUS              RESTARTS   AGE
    kubernetes-bootcamp-2457653786-bvhgb   0/1       ContainerCreating   0          24s
    kubernetes-bootcamp-2457653786-kvqdj   0/1       ContainerCreating   0          24s
    kubernetes-bootcamp-2457653786-mlbb8   1/1       Running             0          52m
    kubernetes-bootcamp-2457653786-txsp5   1/1       Running             0          24s

上の例では、一部のPodは起動中の状態です。少し時間が経過すると全てのPodのSTATUSがRunningになります。

### 4.2. Podへのルーティング
sternで標準出力を確認すると、以下の様にPodが追加されていることがわかります。

    + kubernetes-bootcamp-2457653786-txsp5 › kubernetes-bootcamp
    kubernetes-bootcamp-2457653786-txsp5 kubernetes-bootcamp Kubernetes Bootcamp App Started At: 2017-11-20T03:42:40.704Z | Running On:  kubernetes-bootcamp-2457653786-txsp5
    kubernetes-bootcamp-2457653786-txsp5 kubernetes-bootcamp
    + kubernetes-bootcamp-2457653786-bvhgb › kubernetes-bootcamp
    kubernetes-bootcamp-2457653786-bvhgb kubernetes-bootcamp Kubernetes Bootcamp App Started At: 2017-11-20T03:43:06.276Z | Running On:  kubernetes-bootcamp-2457653786-bvhgb
    kubernetes-bootcamp-2457653786-bvhgb kubernetes-bootcamp
    + kubernetes-bootcamp-2457653786-kvqdj › kubernetes-bootcamp
    kubernetes-bootcamp-2457653786-kvqdj kubernetes-bootcamp Kubernetes Bootcamp App Started At: 2017-11-20T03:43:08.604Z | Running On:  kubernetes-bootcamp-2457653786-kvqdj
    kubernetes-bootcamp-2457653786-kvqdj kubernetes-bootcamp

sternは複数のPodからの標準出力を自動的に色分けして表示します。実際に各Podからの出力が色で識別できることを確認してください。

何度かServiceにアクセスしてみると、複数のPodから応答が帰ってきていることを確認できます。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    curl http://${NODE01}:${NODE_PORT}/

:fa-windows: __Windows__

    > Invoke-RestMethod -Uri "http://${NODE01}:${NODE_PORT}/"


同様にして、Podを2つに縮小することも可能です。

    kubectl scale deployments/kubernetes-bootcamp --replicas=2


---
以上で本チュートリアルは終了です。
