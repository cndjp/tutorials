リクエスト・ルーティングの制御
==============================
このチュートリアルでは、Istioを使ってリクエスト・ルーティングの制御を体験します。カナリーデプロイメントのシナリオを想定して、複数バージョンが混在するアプリケーションのコンテナ群に対して、バージョン毎に流量を変えてルーティングしてみます。


前提条件
--------
- Istioをインストール済みのKubernetesクラスター
- 上記クラスターに対して実行可能なKubernetes、Istioのコマンドラインツール(kubectl,istioctl)

!!! Note
    KubernetesクラスターへのIsitoのインストール手順は、[Istioのインストール](./install-istio.md)を参照してください。

以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。


1. サンプルアプリケーションを配備する
-------------------------------------
このチュートリアルでは、サンプルアプリケーションとしてkubernetes-bootcampアプリケーションを使います。ここでは、リクエスト・ルーティングの制御を行う前に、このアプリケーションをIstioのSidecarなしでデプロイし、動作を確認しておきます。

### 1.1. Namespaceを作成する
以降のチュートリアルの手順で利用するNamespaceとして、"istio-routing"を作成しておきます。

    kubectl create namespace istio-routing

デフォルトNamespaceを"istio-routing"に変更しておくことで、kubectlの実行の度にNamespaceを指定しなくても良いようにしておきます。

    kubectl config set-context $(kubectl config current-context) --namespace=istio-routing

### 1.2. bootcampアプリケーションのデプロイ
このチュートリアルで利用するmanifestファイル一式を``clone``し、出来上がったディレクトリをカレントディレクトリにしておきます。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    git clone https://github.com/cndjp/istio-request-routing && cd istio-request-routing

:fa-windows: __Windows__

    git clone https://github.com/cndjp/istio-request-routing; cd istio-request-routing

続いて、bootcampアプリケーションをデプロイします。

    kubectl create -f ./bootcamp.yaml

Podの一覧を表示して、version 1のPodが2つ、varsion 2のPodが1つデプロイされていることを確認してください。

    kubectl get pods

正しくPodがデプロイされていれば、以下のような結果が表示されます。

    NAME                           READY     STATUS    RESTARTS   AGE
    bootcamp-v1-68cb78f87c-l8szm   1/1       Running   0          42s
    bootcamp-v1-68cb78f87c-zlmqh   1/1       Running   0          42s
    bootcamp-v2-569574d79f-w4p6g   1/1       Running   0          41s

次に、クラスター外からアプリケーションにアクセスできるようにするために、ServiceおよびIngressオブジェクトを作成します。

Serviceオブジェクトを作成するには以下のコマンドを実行します。

    kubectl create -f ./bootcamp-service.yaml

Ingressオブジェクトを作成するには以下のコマンドを実行します。

    kubectl create -f ./bootcamp-ingress.yaml

!!! Note
    ここではIngress Cotrollerとして、Istioに付属のIstio Ingress Cotrollerを利用します。Ingressオブジェクトを作成すると、その設定内容に合わせてIngress ControllerがL7ロードバランサとして機能します。これによって、クラスター外からのリクエストをアプリケーションに届けることができます。


### 1.3. bootcampアプリケーションへのアクセスの確認
Ingress Controller経由でのアプリケーションへのアクセスを試行してみます。

まず、Ingress ControllerのPodが配備されているNodeのIPアドレスと、公開されているPort番号を取得して、GATEWAY\_URLという環境変数に格納しておきます。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    export GATEWAY_URL=$(kubectl get po -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}')

:fa-windows: __Windows__

    $GATEWAY_URL=$(kubectl get po -l istio=ingress -n istio-system -o jsonpath='{.items[0].status.hostIP}') + ':'+ $(kubectl get svc istio-ingress -n istio-system -o jsonpath='{.spec.ports[0].nodePort}')

GATEWAY\_URLを表示して、[IPアドレス]:[ポート番号]の形式で実際の値が格納されていることを確認してみてください。

    echo $GATEWAY_URL

次に、取得したURLの末尾に"/bootcamp"を追加して、その宛先に対してリクエストを送信します。Pod名とアプリケーションのバージョン番号が返却されることを確認してください。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    curl http://$GATEWAY_URL/bootcamp

:fa-windows: __Windows__

    Invoke-RestMethod -Uri "http://$GATEWAY_URL/bootcamp"

この時点では、トラフィックの制御を何も行っていないため、3つのPodに対して均等の割合でルーティングされます。リクエストの送信を繰り返し実行すると、おおよそ2/3がv1、1/3がv2からの応答となります。

以上で、サンプルアプリケーションの配備は完了です。


2. Istioによるリクエスト・ルーティングの制御をおこなう
------------------------------------------------------
それでは、Istioを使ったリクエスト・ルーティングの制御を実際におこなってみます。

### 2.1. bootcampアプリケーションのPodをEnvoyを注入したものに置き換える
ここまでの手順でデプロイしたアプリケーションは、Kubernetesの通常の方法でデプロイしているため、PodにEnvoyプロキシを含んでいません。まずは、これをEnvoyを含むものに置き換えます。

以下のコマンドで、一度bootcampアプリケーションを削除します。

    kubectl delete -f ./bootcamp.yaml

次に、istioctlを使って、bootcampアプリケーションのmanifestファイルにEnvoyの記述を追加します。

    istioctl kube-inject --debug -f ./bootcamp.yaml -o ./bootcamp-istio.yaml

新たに作成された、Envoyプロキシ注入後のmanifestファイルの内容を参照してみます。

    cat ./bootcamp-istio.yaml

以下は、v1のbootcampアプリケーションに当たる部分を抜粋したものです。主要な追記箇所をハイライトしています。

    #!diff
     apiVersion: apps/v1beta1
     kind: Deployment
     metadata:
       creationTimestamp: null
       name: bootcamp-v1
     spec:
       replicas: 2
       selector:
         matchLabels:
           app: bootcamp
       strategy: {}
       template:
         metadata:
    +      annotations:
    +        sidecar.istio.io/status: '{"version":"15bbcd92b59e99a83360c48bd7af9e84ffd5b961b914e8febcee0e38d08b773e","initContainers":["istio-init","enable-core-dump"],"containers":["istio-proxy"],"volumes":["istio-envoy","istio-certs"]}'
           creationTimestamp: null
           labels:
             app: bootcamp
             version: v1
         spec:
           containers:
           - image: docker.io/jocatalin/kubernetes-bootcamp:v1
             name: bootcamp
             ports:
             - containerPort: 8080
             resources: {}
    +      - args:
    +        - proxy
    +        - sidecar
    +        - --configPath
    +        - /etc/istio/proxy
    +        - --binaryPath
    +        - /usr/local/bin/envoy
    +        - --serviceCluster
    +        - bootcamp
    +        - --drainDuration
    +        - 45s
    +        - --parentShutdownDuration
    +        - 1m0s
    +        - --discoveryAddress
    +        - istio-pilot.istio-system:8080
    +        - --discoveryRefreshDelay
    +        - 1s
    +        - --zipkinAddress
    +        - zipkin.istio-system:9411
    +        - --connectTimeout
    +        - 10s
    +        - --statsdUdpAddress
    +        - istio-mixer.istio-system:9125
    +        - --proxyAdminPort
    +        - "15000"
    +        - --controlPlaneAuthPolicy
    +        - NONE
    +        env:
    +        - name: POD_NAME
    +          valueFrom:
    +            fieldRef:
    +              fieldPath: metadata.name
    +        - name: POD_NAMESPACE
    +          valueFrom:
    +            fieldRef:
    +              fieldPath: metadata.namespace
    +        - name: INSTANCE_IP
    +          valueFrom:
    +            fieldRef:
    +              fieldPath: status.podIP
    +        image: docker.io/istio/proxy_debug:0.7.1
    +        imagePullPolicy: IfNotPresent
    +        name: istio-proxy
    +        resources: {}
    +        securityContext:
    +          privileged: true
    +          readOnlyRootFilesystem: false
    +          runAsUser: 1337
    +        volumeMounts:
    +        - mountPath: /etc/istio/proxy
    +          name: istio-envoy
    +        - mountPath: /etc/certs/
    +          name: istio-certs
    +          readOnly: true
    +      initContainers:
    +      - args:
    +        - -p
    +        - "15001"
    +        - -u
    +        - "1337"
    +        image: docker.io/istio/proxy_init:0.7.1
    +        imagePullPolicy: IfNotPresent
    +        name: istio-init
    +        resources: {}
    +        securityContext:
    +          capabilities:
    +            add:
    +            - NET_ADMIN
    +          privileged: true
    +      - args:
    +        - -c
    +        - sysctl -w kernel.core_pattern=/etc/istio/proxy/core.%e.%p.%t && ulimit -c
    +          unlimited
    +        command:
    +        - /bin/sh
    +        image: alpine
    +        imagePullPolicy: IfNotPresent
    +        name: enable-core-dump
    +        resources: {}
    +        securityContext:
    +          privileged: true
    +      volumes:
    +      - emptyDir:
    +          medium: Memory
    +        name: istio-envoy
    +      - name: istio-certs
    +        secret:
    +          optional: true
    +          secretName: istio.default
    +status: {}

追記箇所のポイントを挙げると、以下の2点があります。

- __{.spec.template.spec.containers}__以下に、istio-proxyという名前のコンテナが追加されており、これがEnvoyプロキシの実態となる。また、Envoyプロキシが利用するための環境変数の記述も併せて追加されている
- __{.spec.template.spec.initContainers}__以下に、istio-init, enable-core-dumpという名前のinitContainerが追加されている

!!! Note
    initContainerは、Podのデプロイ時に前処理を行うために利用するコンテナで、これはKubernetesが持っている機能です。

    前処理でしか使わないバイナリをinitContainerに集めておく、前処理でしか必要ない権限はinitContainerだけに与える、などをおこなうことで、本体のContainerを必要最低限の機能に絞り込んだものにすることができます。

    上の例で、istio-initはEnvoyプロキシが利用するためのiptablesのルールの設定をおこなうもの、enable-core-dumpはコアダンプの出力の設定をおこなうものです。


それでは、このmanifestを使って、bootcampアプリケーションをデプロイします。

    kubectl create -f ./bootcamp-istio.yaml

### 2.2. リクエスト・ルーティングの制御を行う
それでは、実際にトラフィックの流量制御を行います。

まずは、全てのトラフィックをv1に振り分けてみます。トラフィックのルールはIstio用のmanifestファイルとして記述することができます。実際のmanifestファイルの内容は以下のようになっています。

```
cat ./bootcamp-route-rule-all-v1.yaml
```

    #!yaml hl_lines="9 10 11 12"
    apiVersion: config.istio.io/v1alpha2
    kind: RouteRule
    metadata:
      name: bootcamp-default
    spec:
      destination:
        name: bootcamp
      precedence: 1
      route:
      - labels:
          version: v1
        weight: 100

__{.spec.route}__に具体的なルーティングのルールを記述しています。この場合、bootcampというServiceがルーティング対象とするPodで、version: v1というlabelが設定されているものに、全てのトラフィックを送るようにしています。

このルールを適用するには、以下のコマンドを実行します。

    istioctl create -n istio-routing -f ./bootcamp-route-rule-all-v1.yaml

改めて、アプリケーションにリクエストを送信します。繰り返し実行してみて、全ての応答がv1から返却されることを確認してください。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    curl http://$GATEWAY_URL/bootcamp

:fa-windows: __Windows__

    Invoke-RestMethod -Uri "http://${GATEWAY_URL}/bootcamp"

次に、v1,v2にトラフィックを均等に送るようにルールを変更します。
まずは、新たにデプロイするルールのmanifestファイルを参照してみます。

```
cat ./bootcamp-route-rule-50-v2.yaml
```

    #!yaml hl_lines="9 10 11 12 13 14 15"
    apiVersion: config.istio.io/v1alpha2
    kind: RouteRule
    metadata:
      name: bootcamp-default
    spec:
      destination:
        name: bootcamp
      precedence: 1
      route:
      - labels:
          version: v1
        weight: 50
      - labels:
          version: v2
        weight: 50

このmanifestでは、v1,v2それぞれにweight: 50を記述することで、それぞれのバージョンに均等にトラフィックが送信されるようにしています。

それでは、このルールをデプロイします。以下のような``istioctl replace``コマンドをを使ってルールの置き換えをおこないます。

    istioctl replace -n istio-routing -f ./bootcamp-route-rule-50-v2.yaml

再度リクエストの送信を繰り返して、v1とv2からの応答がおおよそ同じ割合で返ってくることを確認してください。

最後に、v2に全てのトラフィックをルーティングするように、ルールを変更します。

    istioctl replace -n istio-routing -f ./bootcamp-route-rule-all-v2.yaml

今度は、応答が全てv2からのものになっていることを確認してください。


3. クリーンアップ
-----------------
これまで作ってきたオブジェクトをクリーンアップしたい場合は、以下のコマンドを実行して、作成したオブジェクトを削除してください。

```
istioctl delete -n istio-routing -f ./bootcamp-route-rule-all-v2.yaml
```
```
kubectl delete namespace istio-routing
```

---
以上で本チュートリアルは終了です。

