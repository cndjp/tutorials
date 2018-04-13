minikube
========
minikubeを利用して、Kubernetesクラスターを構成します。minikubeはKubernetesの公式GitHubグループ配下で管理されているツールで、ローカルPC上に立ち上げた仮想マシン上に、Kubernetesのシングルノードクラスターを構成します。

ここでは、Windows及びMacOS Xで、仮想マシンのハイパーバイザーとしてVirtualBoxを使う場合の、minikubeのインストール手順を記します。併せて、Kubernetesを操作するためのコマンドラインツールである、kubectlのセットアップも行います。

使用するPCのOSに合わせて、以下のリンクから手順に進んでください。

- [:fa-windows: Windows](#1-windows-pcminikube)
- [:fa-apple: MacOS X](#2-macos-xminikube)


1. :fa-windows: Windows PCでのminikubeのインストール手順
--------------------------------------------------------
ここでは、[chocolatey](https://chocolatey.org/)（Windowsのパッケージマネージャ）を使ったインストール手順を記します。

### 前提条件

- Windows 10 64bit
- 管理者権限での操作が可能なWindowsユーザーアカウントでログオンしていること


### 1.1. chocolateyのインストール
管理者権限でWindows PowerShellを起動し、以下のコマンドを実行してください。

    Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

プロンプトに表示されるメッセージに従ってインストールを進めます。

以降はchocolateyを使って順次必要なものをインストールしていきます。ご自身の環境にインストールされていないものについて、以下の手順を実施してください。

### 1.2. Virtual Boxのインストール

    choco install virtualbox

### 1.3. kubectlのインストール
kubectlをインストールするには、以下のコマンドを実行します。

    choco install kubernetes-cli

### 1.4. minikubeのインストール
minikubeをインストールするには、以下のコマンドを実行します。

    choco install minikube

以下のコマンドでminikubeがインストールされていること確認してください。

    minikube version

    minikube version: v0.25.x


2. :fa-apple: MacOS Xでのminikubeのインストール手順
--------------------------------------------------------
（工事中）


3. minikubeクラスターの操作方法
-------------------------------

minikubeクラスターの起動/停止/削除をするには、以下のコマンドを実行します。

##### 起動

    > minikube start

##### 停止

    > minikube stop

##### 削除

    > minikube delete

---
