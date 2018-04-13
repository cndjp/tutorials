minikube
========
ここでは、minikubeを利用してKubernetesクラスターを構成します。minikubeはKubernetesの公式GitHubグループ配下で開発されているツールで、ローカルPCに立ち上げた仮想マシン上に、Kubernetesのシングルノードクラスターを構成することができます。

ここでは、Windows及びMacOS Xで、仮想マシンのハイパーバイザーとしてVirtualBoxを使う場合の、minikubeのインストール手順を記します。併せて、Kubernetesを操作するためのコマンドラインツールである、kubectlのセットアップも行います。

使用するマシンのOSに合わせて、以下のリンクから手順に進んでください。

- [:fa-windows: Windows](#1-windows-pcminikube)
- [:fa-apple: MacOS X](#2-macos-xminikube)（準備中）


1. Windows PCでのminikubeのインストール手順
-------------------------------------------
Windows PC上にminikubeをインストールする手順を記します。ここでは、Windowsのパッケージマネージャである、[chocolatey](https://chocolatey.org/)を使ってインストールをおこないます。

### 前提条件

- Windows 10 64bit
- 管理者権限を持ったWindowsユーザーアカウントでログオンしていること

### 1.1. chocolateyのインストール
管理者権限でWindows PowerShellを起動し、以下のコマンドを実行します。

    Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

インストールプロセスの間、以下のような内容で始まるメッセージが表示されます。インストールが完了するまで1分程度待ちます。

    Getting latest version of the Chocolatey package for download.
    Getting Chocolatey from https://chocolatey.org/api/v2/package/chocolatey/0.10.10.
    …

インストールが完了したら以下のコマンドでバージョン情報を表示し、chocolateyを実行できることを確認します。

    choco -v

バージョン情報は以下のように表示されます。

    0.10.10

以上でchocolateyのインストールは完了です。

!!! Note
    chocolateyでソフトウェアをインストールするには、必ず管理者権限で立ち上げたPowerShell上で実行する必要があります。以降の手順も同様ですのでご注意ください。

### 1.2. VirtualBoxのインストール
chocolateyでVirtualBoxをインストールします。以下のコマンドを実行してください。

    choco install virtualbox

途中、インストール続行の確認のメッセージが表示されるので、++y+enter++を入力します。

    Do you want to run the script?([Y]es/[N]o/[P]rint): y

以下のようなメッセージが表示されたら、インストール完了です。

    Chocolatey installed 1/1 packages.
     See the log for details (C:\ProgramData\chocolatey\logs\chocolatey.log).

### 1.3. kubectlのインストール
同じくchocolateyでkubectlをインストールします。以下のコマンドを実行してください。

    choco install kubernetes-cli

途中、インストール続行の確認のメッセージが表示されるので、++y+enter++を入力します。

    Do you want to run the script?([Y]es/[N]o/[P]rint): y

インストールが完了したら、以下のコマンドでバージョン情報を表示し、kubectlを実行できることを確認します。

    kubectl version

バージョン情報は以下のように表示されます。

    Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.1", GitCommit:"d4ab47518836c750f9949b9e0d387f20fb92260b", GitTreeState:"clean", BuildDate:"2018-04-12T14:26:04Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"windows/amd64"}
    Unable to connect to the server: dial tcp [::1]:8080: connectex: No connection could be made because the target machine actively refused it.

### 1.4. minikubeのインストール
ここでもchocolateyを使います。minikubeをインストールするには以下のコマンドを実行します。

    choco install minikube

インストールが完了したら、以下のコマンドでバージョン情報を表示し、kubectlを実行できることを確認します。

    minikube version

バージョン情報は以下のように表示されます。

    minikube version: v0.26.0

以上で、Windows PCでのminikubeのインストールは完了です。[3. minikubeによるKubernetesクラスターの操作](#3-minikubekubernetes)の手順に進んでください。


2. MacOS Xでのminikubeのインストール手順
----------------------------------------
（準備中）


3. minikubeによるKubernetesクラスターの操作
-------------------------------------------
ここでは、minikubeでクラスターの起動／停止などの基本的な操作をおこなってみます。

### 3.1. クラスターの起動と接続確認
minikubeクラスターを起動するには、以下のコマンドを実行します。

    minikube start

Windowsでは、初回の実行時にユーザーアカウント制御のダイアログが表示される場合があります。その場合、コンピューターへの操作を許可する旨のボタンをクリックしてください。

``minikube start``コマンドを実行すると、kubectlに対して自動的にminikubeクラスターへの接続設定がおこなわれます。以下のコマンドを実行して、kubectlでクラスターのNodeの情報が取得できることを確認してください。

    kubectl get nodes

クラスターに接続できていれば、以下のような結果が表示されます。

    NAME       STATUS    ROLES     AGE       VERSION
    minikube   Ready     <none>    7h        v1.9.0

また、``minikube status``を実行すると、クラスターの起動状態を確認できます。

    minikube status

クラスタの起動中に上記コマンドを実行すると、以下のような結果が表示されます。

    minikube: Running
    cluster: Running
    kubectl: Correctly Configured: pointing to minikube-vm at 192.168.39.52

### 3.2. クラスターの停止とクリーンアップ
minikubeクラスターを停止するには、以下のコマンドを実行します。

    minikube stop

クラスターが停止した状態で``minikube start``を実行するとクラスターを再起動できます。また、クラスターを仮想マシンごと削除してクリーンアップしたい場合には、``minikube delete``を実行します。

    minikube delete

---
以上で、minikubeによるKubernetesクラスターの構築、及びkubectlのセットアップは完了です。
