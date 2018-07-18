Web-AP-DBの3層アプリケーションが動くまで
========================================
!!! Warning
    本チュートリアルはまだ執筆作業中ですので、記述どおりに作業を行っても動作しない可能性があります。

このチュートリアルでは、Kubernetesクラスター上に、node.js/MySQLを使ったアプリケーション(koa-sample)をデプロイします。外部からのリクエストを受けるロードバランサーの代わりとして、KubernetesのIngressを利用します。


前提条件
--------
- 任意のKubernetesクラスター
- 上記Kubernetesクラスターに接続可能なkubectl

!!! Note
    本チュートリアルで、[ローカルPCにKubernetesクラスターを構成する手順](../../index.md#appendix)もご案内していますので、適宜ご利用ください。

!!! Note
    以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。


0 . 準備作業
------------
まずは、このチュートリアルを進めるために必要なファイル一式の準備と、作業領域として利用するNamespaceを作成します。

### 0.1. Namespaceを作成する
以降のチュートリアルの手順で利用するNamespaceとして、"koa-sample"を作成しておきます。

    kubectl create namespace koa-sample

デフォルトNamespaceを"bootcamp"に変更しておくことで、kubectlの実行の度にNamespaceを指定しなくても良いようにしておきます。

    kubectl config set-context $(kubectl config current-context) --namespace=koa-sample

アプリケーションの一連のmanifestファイルがこのリポジトリに保存されています。まずはこのリポジトリを適当なディレクトリにcloneして下さい。

以下は、コマンドラインツールのgitを使う例です。

    git clone https://github.com/oracle-japan/cndjp2.git

cloneしてできたディレクトリ配下の、koa-sampleをカレント・ディレクトリにしておきます。

    cd cndjp2/koa-sample

このディレクトリ配下に、アプリケーションを動かすために必要なファイル一式が格納されています。


1 . MySQLデータベースを動かす
-----------------------------

### 1-1. MySQLのDockerイメージの準備
MySQLをKubernetesを動かすためには、予めコンテナイメージを作成してコンテナレジストリにアップロードしておく必要があります。
今回は既にこの手順を実施済みですが、以下に作業内容を紹介します。

このチュートリアルで利用するDBのコンテナイメージは、cndjp2/db配下のファイル群を使って作成してあります。

    > ls db
    create_schema.sql  Dockerfile

Dockerfileを参照すると、Docker Hub公式のMySQLイメージを使っており、所定のフォルダに.sqlファイルを配置していることがわかります。

    > cat db/Dockerfile
    FROM mysql:8.0.3
    
    COPY ./create_schema.sql /docker-entrypoint-initdb.d/

公式のMySQLイメージは、/docker-entrypoint-initdb.d/に.sqlファイルを配置しておくと、イメージの起動時にその.sqlを実行してくれるようになっています。また、イメージの起動時に所定の環境変数を設定すると、必要なDBユーザーを作成するなどが可能です。<br>
詳細は[公式イメージのリポジトリのページ](https://hub.docker.com/_/mysql/)を参照して下さい。

今回は、アプリケーションで利用するスキーマとデータを、この仕組みを利用して作成することにしています。

このようなDockerfileをつかって作成したイメージを、[Docker Hubに登録](https://hub.docker.com/r/cndjp/koa-sample-db/)してあります。

### 1-2. DB用のPersistentVolumeの作成
次に、MySQLデータベース用のデータ保存領域として利用するための、PersistentVolumeを作成します。

このためのmanifestファイルを作成してありますので、中身を参照してみて下さい。

    > cat deployment/db-pv-hostpath.yaml

内容は以下のとおりです。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-hostpath-pv
  labels:
    type: local
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/mysql
    # type: DirectoryOrCreate
```

この例では、hostPathのVolumeを作成しています。hostPathはNodeのローカルボリュームを利用する方法で、Node上のファイルにアクセスする必要がある場合を除くと、本番用途に適したものではありません。<br>
本番向けには、該当箇所を適切に変更して、別のVolumeを利用する必要があります。

それでは、このPersistentVolumeをクラスター上に作成します。

    > kubectl create -f deployment/db-pv-hostpath.yaml
    persistentvolume "mysql-hostpath-pv" created

PersistentVolume一覧を取得する際は、以下のコマンドを実行します。

    > kubectl get pv

PersistentVolumeの詳細情報は以下のように取得します。

    > kubectl describe pv/mysql-hostpath-pv

### 1-3. データベースユーザーのSecretの作成
データベースユーザーの情報は、DBを稼働させるコンテナのmanifestに指定する事もできますが、その方法では、manifestファイル内に直接パスワードを記述することになってしまいます。これを避けるため、データベースユーザーのユーザー名/パスワードを保存するSecretオブジェクトを作成して、これを参照するようにします。

このためのmanifestファイルも以下に作成済みです。

    > cat deployment/db-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dbsecret
type: Opaque
data:
  root_password: bXlzcWw=
  user: a29h
  password: a29h
```

Secretに保存したいデータは、`data`配下に、Key/Valueの形式で記述します。このとき、値をbase64でエンコードしておく必要があります。

__注意__: Secretのmanifestをソースコードリポジトリに保存してしまうと、パスワード等の重要な情報がリポジトリを通して露出してしまいます。実際の運用では、このファイルまたはファイルに記述する値は、別途管理する必要があります。

<!-- ここでSecretを作らせてみる -->

それでは、このSecretをクラスター上に作成します。

    > kubectl create -f deployment/db-secret.yaml
    secret "dbsecret" created

作成されたSecretを確認してみます。

・一覧取得

    > kubectl get secrets

・詳細情報取得

    > kubectl describe secret/dbsecret

### 1-4. データベースのコンテナの起動
データベースのコンテナをデプロイします。ポイントは、データベース用のデータ保存領域としてPersistentVolumeを利用することと、Secretにあるユーザー名/パスワードを環境変数として指定してコンテナを起動することです。

    > cat deployment/db-deployment.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: koa-sample-db
spec:
  ports:
  - port: 3306
  selector:
    app: koa-sample-db
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1beta1 # for versions before 1.8.0 use apps/v1beta1
kind: Deployment
metadata:
  name: koa-sample-db
spec:
  selector:
    matchLabels:
      app: koa-sample-db
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: koa-sample-db
    spec:
      containers:
      - image: cndjp/koa-sample-db:0.1
        name: koa-sample-db
        env:
        - name: MYSQL_ROOT_PASSWORD
          # value: mysql
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: root_password
        - name: MYSQL_USER
          # value: koa
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: user
        - name: MYSQL_PASSWORD
          # value: koa
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: password
        ports:
        - containerPort: 3306
          name: koa-sample-db
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

このmanifestでは3つのオブジェクトを作成しています。PersistetVolumeを確保するためのPersistentVolumeClaim、DBをクラスター内に公開するためのService、DB本体を動かすPod(Deployment)です。

特にDB本体のDeploymentにおいて、環境変数としてSecretの値を参照しているところを探してみて下さい。

それでは、これらのオブジェクトをデプロイします。

    > kubectl create -f deployment/db-deployment.yaml
    service "koa-sample-db" created
    persistentvolumeclaim "mysql-pv-claim" created
    deployment "koa-sample-db" created

Podの一覧を表示すると始めのうちはSTATUSがContainerCreatingになっています。

    > kubectl get pods
    NAME                             READY     STATUS              RESTARTS   AGE
    koa-sample-db-3605205203-rq81f   0/1       ContainerCreating   0          30s

少しだけ時間をおいて、Runningになることを確認して下さい。

ここで、実際にデータベースに繋いでみます。<br>
データベースはServiceオブジェクトを通して公開されていますが、clusterIPモードでServiceを作成しているため、クラスター内からしかアクセスできません。そこで、クラスター内にMySQLのクライアント専用のコンテナをデプロイして、それ経由でアクセスします。

    > kubectl run -it --rm --image=mysql:8.0.3 --restart=Never mysql-client -- mysql -h koa-sample-db -pmysql
    If you don't see a command prompt, try pressing enter.
    
    mysql>

MySQLクライアントのプロンプトで、以下のコマンドを実行して、所定のデータベースとユーザーが作成されていることを確認して見て下さい。

    mysql> show databases;

    mysql> select Host, User from mysql.user;


2 . アプリケーションを動かす
----------------------------

### 2-1. Node.jsアプリケーションのDockerイメージの準備
データベース同様、アプリケーションを予めコンテナイメージ化して、コンテナレジストリに保存しておく必要があります。

今回は既にこの手順を実施済みですが、以下に作業内容を紹介します。

このチュートリアルで利用するアプリケーションのコンテナイメージは、cndjp2/ap配下のファイル群を使って作成してあります。

    > ls ap
    Dockerfile  README.md   app-api/  app.js  logs/    package.json  test/
    LICENSE     app-admin/  app-www/  lib/    models/  public/

こちらもDocker Hub公式のNode.jsイメージをベースとして使っており、所定のフォルダにアプリケーションのフォルダを配置して、必要なセットアップを行っています。

    > cat ap/Dockerfile
    FROM node:9.2.1
    
    ENV APP_ROOT /usr/src/koa-sample-ap
    
    COPY . $APP_ROOT
    WORKDIR $APP_ROOT
    
    RUN npm install && npm cache verify
    
    EXPOSE 3000
    
    CMD ["npm", "start"]

公式のNode.jsイメージの詳細は[公式イメージのリポジトリのページ](https://hub.docker.com/_/node/)を参照して下さい。

このようなDockerfileをつかって作成したイメージを、[Docker Hubに登録](https://hub.docker.com/r/cndjp/koa-sample-ap/)してあります。

### 2-2. アプリケーションのコンテナのデプロイ
アプリケーションのコンテナをデプロイします。ポイントは、データベースのアクセスユーザー上納を、Secretから取得するように記述することです。<br>
DB、アプリともユーザー情報をSecretから取得することで、ユーザー名/パスワードを変更したいときSecretを変更しさえすればよいようにできます。

    > cat deployment/ap-deployment.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: koa-sample-ap
spec:
  selector:
    app: koa-sample-ap
  ports:
  - port: 3000
  clusterIP: None
---
apiVersion: apps/v1beta1 # for versions before 1.8.0 use apps/v1beta1
kind: Deployment
metadata:
  name: koa-sample-ap
spec:
  replicas: 2
  selector:
    matchLabels:
      app: koa-sample-ap
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: koa-sample-ap
    spec:
      containers:
      - image: cndjp/koa-sample-ap:0.1
        name: koa-sample-ap
        env:
          # Use secret in real usage
        - name: DB_HOST
          value: koa-sample-db
        - name: DB_PORT
          value: "3306"
        - name: DB_USER
          # value: koa
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: user
        - name: DB_PASSWORD
          # value: koa
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: password
        - name: DB_DATABASE
          value: koa-sample-sandbox
        ports:
        - containerPort: 3000
          name: koa-sample-ap
```

ここでも、データベースユーザーのユーザー名/パスワードをSecretから取得して、環境変数に与えている箇所を確認してみて下さい（環境変数にユーザー名、パスワードを設定するのは、このサンプルアプリの仕様です）。

また、このアプリもclusterIPモードのServiceを通して公開されており、クラスター内からしかアクセスすることはできません。<br>
クラスター外部への公開は、次章以降でIngressを使って行います。

それでは、このアプリケーションをデプロイします。

    > kubectl create -f deployment/ap-deployment.yaml
    service "koa-sample-ap" created
    deployment "koa-sample-ap" created

ここでもPodのステータスがRunningになるまで、少し時間を置いて下さい

    > kubectl get pods
    NAME                             READY     STATUS              RESTARTS   AGE
    koa-sample-ap-4193452636-03r9s   0/1       ContainerCreating   0          9s
    koa-sample-ap-4193452636-13vnj   0/1       ContainerCreating   0          9s
    koa-sample-db-3605205203-rq81f   1/1       Running             0          36m


3 . Ingressを使ったロードバランサーの構成
-----------------------------------------

### 3-1. IngressとNGINX Ingress Controllerのデプロイ
Ingressを使うと、外部アクセスのロードバランシングやSSL/TLSの終端といった、Webフロントの機能を、クラスター内に構成することができます（IngressはKubernetes 1.8の時点ではBetaの機能です。現時点では、クラスター外にLBを構成する方法が主流となっています）。

Ingressオブジェクトは、Webフロントとしての構成情報を設定するオブジェクトで、実際の機能を担う実態は、別途ReplicationControllerとしてデプロイする必要があります。
今回はNGINXを利用した、[NGiNX Ingress Controller](https://github.com/kubernetes/ingress-nginx)を使います。

まずはじめに、外部からのリクエストに対してルーティング先が見つからなかったときのレスポンスを返すものとして、デフォルトのバックエンドを構成しておきます。

    > kubectl create -f deployment/web-default-backend.yaml
    replicationcontroller "default-http-backend" created

    > kubectl expose rc default-http-backend --port=80 --target-port=8080 --name=default-http-backend
    service "default-http-backend" exposed

今回のNGINX Ingress Controllerのmanifestでは、role=frontというLabelのついたNodeにデプロイされるように設定しています。以下のようにNodeにLabelを設定して、デプロイ先のNodeが固定されるようにします。

    > kubectl label nodes 172.17.8.102 role=front

NGINX Ingress Controllerをデプロイします。

    > kubectl create -f deployment/web-rc-default.yaml
    replicationcontroller "nginx-ingress-controller" created

最後にIngressオブジェクトを構成します。

    > kubectl create -f deployment/web-ingress.yaml


### 3-2. アプリケーションへのアクセス
まず、hostsファイルを編集し、アプリケーションにアクセスするためのエントリーを追加します。
プラットフォーム毎のhostsファイルの場所は、以下のとおりです。

・Linux

    > /etc/hosts

・Mac
    > /private/etc/hosts

・Windows

    > C:\Windows\System32\drivers\etc\hosts"
<!--
    > powershell -NoProfile -ExecutionPolicy unrestricted -Command "start notepad C:\Windows\System32\drivers\etc\hosts -verb runas"
-->

以下のエントリーを追加して下さい。

    172.17.8.102        k8s
    172.17.8.102        admin.k8s
    172.17.8.102        api.k8s
    172.17.8.102        www.k8s

サンプルアプリケーションのAPIにアクセスするには、以下のコマンドを実行します。これは、アプリケーションにログインするREST APIを実行しています。

    > curl -X GET "http://api.k8s/auth?username=admin@user.com&password=admin"

返却されたJWTのトークン（ランダムな文字列）をコピーしておきます。

つづいて、以下のコマンドの[JWT Token]部分を上記のトークンに置き換えて、コマンドを実行します。

    > curl -X GET -H "Authorization: Bearer [JWT Token]" http://api.k8s/members

例えば、以下のようなコマンドになります。

    > curl -X GET -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MTAwMDAyLCJyb2xlIjoiYSIsImlhdCI6MTUxMzUzOTk2OCwiZXhwIjoxNTEzNjI2MzY4fQ.pbeXUSE41BHEn1jmd7JaDhose7lWojP_P0zrSCXBx8c" http://api.k8s/members

うまくいけば、以下のように、アプリケーションに登録されているメンバー情報がJSONで取得できます。

    [{"_id":100002,"_uri":"/members/100002"},{"_id":100001,"_uri":"/members/100001"},{"_id":100004,"_uri":"/members/100004"},{"_id":100003,"_uri":"/members/100003"}]

ご自身のPCにクラスターを作成して動かしている方（一時借し出し環境でない方）は、ブラウザで以下のURLにアクセスすると、GUIのアプリケーション画面にアクセスできます。

    http://k8s/

---
ハンズオンのPart2は以上です。

<!--
4. 練習問題
-->
<!--
-->
<!--
余力のある方は、アプリケーションレイヤーのカナリーデプロイメントに挑戦してみて下さい。

カナリーデプロイメントを行うには、LabelとLabelSelectorを利用します。サンプルとして以下の公式ドキュメントの技術を参考にして下さい。

    - https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments
-->
