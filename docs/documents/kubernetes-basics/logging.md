Kubernetesにおけるログの収集
============================
このチュートリアルでは、Kubernetesクラスター上で動作するコンテナのログの、確認、収集をおこなう手順を体験します。

Kubernetesでは、コンテナのログは標準出力に出力するのが基本です。はじめに、kubectlを使ってクラスター内のコンテナの標準出力を見る方法を、次に、2系統以上のログを分けて出力したい場合に、サイドカーコンテナを利用してそれらを分けて出力する手順を試してみます。


前提条件
--------
- Kubernetesクラスター 1.7.x+
- 上記Kubernetesクラスターに接続可能なkubectl

!!! Note
    本チュートリアルで、[ローカルPCにKubernetesクラスターを構成する手順](../../index.md#appendix)もご案内していますので、適宜ご利用ください。

!!! Note
    以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。


1. コンテナのログ（標準出力）を確認する
---------------------------------------
ここでは、kubectlを使ってクラスター内のコンテナの標準出力を表示する方法を試してみます。

### 1.1. 最初のmanifestファイルを作成する
始めに、このチュートリアルの作業をおこなう場所として、適当なパスにk8s-loggingという名前でディレクトリを作成し、それをカレントディレクトリにしておきます。以下は、ホームディレクトリ配下にk8s-loggingを作成する例です。

:fa-apple: __Mac__ / :fa-linux: __Linux__

```sh
mkdir ~/k8s-logging && cd ~/k8s-logging
```

:fa-windows: __Windows__

```bat
mkdir $env:USERPROFILE\k8s-logging; cd $env:USERPROFILE\k8s-logging
```

今回利用するサンプルアプリケーションは、manifestファイルに定義を記述した上で、それを利用してデプロイします。まず、counter.ymlという名前で、空のテキストファイルを作成します。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    touch counter.yml

:fa-windows: __Windows__

    New-Item counter.yml -itemType File

このファイルを適当なエディタで開き、以下の内容をコピー/ペーストし、保存します。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args: [/bin/sh, -c, 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
```

このmanifestファイルでは、counterとい名前のPodを定義しています（以下、このPodをcounterと表記します）。Podが内包するコンテナは、__{.spec.containers}__の配下に記述されており、この例ではcountという名前のひとつのコンテナを含んでいます。

countコンテナは、[busybox](https://hub.docker.com/_/busybox/)というDockerイメージを使っており、ここでは日時を定期的に標準出力に出力するよう記述されています。

### 1.2. Podをデプロイする
manifestファイルを使ってKubnernetesにオブジェクトをデプロイするには、目的のmanifestファイルを入力として、`kubectl create`を実行します。上で作成したcounterのmanifestからデプロイするため、以下のコマンドを実行してください。

    kubectl create -f counter.yml

kubectlでおこなった指示をKubernetesのマスターノードが受け付けると、以下のようなメッセージが返却されます。

    pod "counter" created

Podの一覧を表示して、STATUSがRunningとなっていることを確認してください。

```
kubectl get pods
```
```
NAME      READY     STATUS    RESTARTS   AGE
counter   1/1       Running   0          5s
```

!!! Note
    ``kubectl create``実行時のメッセージは、コンテナが正常に稼働し始めたことを保証するものではありません。メッセージはマスターノードが管理するメタデータが更新されたことをもって返却されるもので、コンテナイメージをクラスター内に取得する、コンテナを起動する、などの処理はこの後におこなわれます。

### 1.3. コンテナの標準出力を確認する
Podの中のコンテナが動き始めると、日時を定期的に標準出力に書き出します。`kubectl logs`コマンドを使うと、その内容を確認できます。

以下の例は、counterの内包するコンテナの過去の標準出力を取得します。

```
kubectl logs counter
```
```
0: Sun Dec 17 14:31:45 UTC 2017
1: Sun Dec 17 14:31:46 UTC 2017
2: Sun Dec 17 14:31:47 UTC 2017
…
```

上記の例では、過去の標準出力を全てコンソールに表示しますが、これを時間や件数で絞ることができます。時間で絞る場合には``--since``を使います。以下の例は、過去10秒の内容に絞って表示します。

    kubectl logs --since 10s counter

件数で絞る場合は``--tail``を使います。以下の例は、過去20件に絞って表示します。

    kubectl logs --tail 20 counter

標準出力を継続して表示し続ける（UNIX/Linux系OSの``tail``コマンドに相当する動作）場合には、-f(follow)オプションを設定します。

    kubectl logs -f counter

``-f``と``--since``や``--tail``を組み合わせて実行することもできます。以下の例では、コマンド実行時点から過去10秒の標準出力を表示し、さらにその後の標準出力を継続して表示し続けます。

    kubectl logs -f --since 10s counter

!!! Note
    コンテナが標準出力に出力した内容は、Docker Engineのlogging driverがリダイレクトします。多くのKubernetesクラスターにおいては、所定のログファイルにリダイレクトするようにlogging driverが構成されています。
    ``kubectl logs``コマンドは、logging driverによって作成されたログファイルの内容を表示していることになります。

### 1.4. counterを削除する
counter削除するには、`kubectl delete`を使います。Podの名前を指定して削除を実行することもできますが、作成時と同じようにmanifestを指定することもできます。以下のコマンドを実行して、counterを削除します。

    kubectl delete -f counter.yml

Kubernetesのマスターノードが指示を受け付けると、以下のようなメッセージが返却されます。

    pod "counter" deleted

すぐにPodの一覧を表示するとSTATUSがTerminatingとなっており、Podを削除中であることがわかります。

    NAME      READY     STATUS        RESTARTS   AGE
    counter   1/1       Terminating   0          25m

しばらくおいてから再びPodの一覧を表示すると、以下のようなメッセージが返却されます。こうなっていれば、Podの削除は完了です。

    No resources found.


2. 2系統のログを出力するコンテナのログの確認方法
------------------------------------------------
2系統以上のログを出力する場合、系統毎にコンテナ用意して同じPodに含めるようにします（sidecarコンテナ）。そして、各sidecarコンテナは、アプリケーションが出力したログファイルで自身が担当する系統のものを、標準出力に書き出すようにします。

ここでは、このようなPodの簡単な例を実際に動かしてみます。


### 2.1. サンプルアプリケーションをデプロイする
まず、two-files-logging-counter.ymlという名前で、空のテキストファイルを作成します。

:fa-apple: __Mac__ / :fa-linux: __Linux__

    touch two-files-logging-counter.yml

:fa-windows: __Windows__

    New-Item two-files-logging-counter.yml -itemType File

このファイルを適当なエディタで開き、以下の内容をコピー/ペーストし、保存します。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: two-files-logging-counter
spec:
  containers:
  - name: two-files-logging-count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

__{.spec.containers}__配下に、このmanifestファイルで定義されたPodが含むコンテナの情報が記述されています。このPodには、two-files-logging-countという名前の2系統のログを出力するコンテナと、count-log-1、count-log-2という名前のsidecarコンテナが含まれています。

two-files-logging-countは、/var/log/1.log、/var/log/2.logという2ファイルに、フォーマットの異なる日時情報を出力しています。2つのsidecarコンテナcount-log-1、count-log-2は、それぞれ/var/log/1.log、/var/log/2.logをtailして標準出力に出力しています。

``kubectl logs``などでこれらのsidecarコンテナの標準出力にアクセスすることっで、two-files-logging-countが出力するログファイルの内容を確認することができます。

それでは、このPodをデプロイします。

    kubectl create -f two-files-logging-counter.yml

Kubernetesのマスターノードが指示を受け付けると、以下のようなメッセージが返却されます。

    pod "two-files-logging-counter" created

数秒〜数十秒程度待ってからPodの情報を見てみます。まず、Podの一覧を表示してみます。

    kubectl get pods

3つのコンテナがREADYとなっていることがわかります。

    NAME                        READY     STATUS              RESTARTS   AGE
    two-files-logging-counter   3/3       Running             0          9s

次にPodの詳細情報を表示してみます。

    kubectl describe pod/two-files-logging-counter

Containers:以下に、Podに含まれるコンテナの情報が表示されており、two-files-logging-countと2つのsidecarコンテナがあることがわかります。

    …
    Containers:
      two-files-logging-count:
        Container ID:       docker://a6a0da9d3625cfb974e5463401f54960b38a8ad345ddd558cfd1bb8a8166250e
        …
      count-log-1:
        Container ID:       docker://389e52693f44642980538744daf79533885d286b69840d0b2f58f1a0b9711836
        …
      count-log-2:
        Container ID:       docker://02702bb46b05414b9d525fd8eada87ceb9a986634e5e43b1f1f138dcbe4e156b
    …

!!! Note
    sidecarコンテナはコンテナを使った設計におけるデザインパターンのひとつです。アプリケーションにそれをサポートする機能を追加したいような場合に、よく利用されます。

    メインのアプリケーションとファイルやネットワークのリソースを共有しながら、独立した別のコンテナとしてsidecarを配置できるため、アプリケーションに影響を与えずに機能拡張するときに便利です。

    sidecarコンテナについては、[Azure アーキテクチャ センター > サイドカー パターン](https://docs.microsoft.com/ja-jp/azure/architecture/patterns/sidecar)に詳しい解説がありますので、参考としてください。

### 2.2.コンテナの標準出力を確認する
sidecarコンテナが2系統のログをそれぞれ標準出力に出力していますので、それぞれのコンテナについて出力内容を確認してみます。

Pod内の個別のコンテナのログを確認するには、以下のようにしてコンテナを指定して``kubectl logs``を実行します。

    kubectl logs [Pod名] [Container名]

count-log-1の標準出力を確認したい場合、以下のようなコマンドとなります。

    kubectl logs two-files-logging-counter count-log-1

同様に、count-log-2の標準出力も表示して、フォーマットの異なる日時情報が出力されていることを確認してみてください。

ここまでは、複数のコンテナの標準出力を個別に確認しましたが、[Stern](https://github.com/wercker/stern)というツールを利用すると、これらを同時にtailしつつ、コンテナ毎に色分けして表示することができ、便利です。

!!! Note
    Sternのインストール手順は、[本チュートリアルの「最初のアプリケーションを動かしてみる」](play-with-bootcamp-app.md#23-podtail)を参照ください。

Sternを今回のPodを対象にして実行してみます。

    stern two-files-logging-counter

2つのsidecarコンテナの標準出力が、色分けされてtailされるこを確認してください。

!!! Note
    sidecarコンテナとしてfluentdを利用するなどして、外部のログ収集基盤にログを転送することもできます。これを実施する際のサンプルとしては、Kubernetesの公式のドキュメントから以下を参考にしてみてください。

    - [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
    - [Logging Using Stackdriver](https://kubernetes.io/docs/tasks/debug-application-cluster/logging-stackdriver/)


### 2.3. two-files-logging-counterを削除する
two-files-logging-counterを削除するには、以下のコマンドを実行します。

    kubectl delete -f two-files-logging-counter.yml

しばらくおいてから再びPodの一覧を表示すると、以下のようなメッセージが返却されます。こうなっていれば、Podの削除は完了です。

    No resources found.

---
以上で本チュートリアルは終了です。
