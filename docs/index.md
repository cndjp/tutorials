Cloud Native Developers JP のハンズオンチュートリアルへようこそ
===============================================================
これは[Cloud Native Developers JP 勉強会](https://cnd.connpass.com/)のハンズオンチュートリアルです。

Kubernetesを中心としたCloud NativeなOSSを、手を動かしながら学べるコンテンツを提供します。勉強会に参加された方もそうでない方も、ご自由に活用ください。


チュートリアルの始め方
----------------------
[Contents](#contents)にあるどの項目からでも始められます。お好きなものをクリックして開始してください。
必要な環境要件は、各チュートリアルの冒頭に記載してあります。


Contents
--------

#### Kubernetesの基礎

- [最初のアプリケーションを動かしてみる](documents/kubernetes-basics/play-with-bootcamp-app.md) :fa-external-link:
:   簡単なサンプルアプリケーションを使って、Kubernetes上へのコンテナのデプロイ、スケールアウト、スケールイン等のオペレーションを体験します。

- [Kubernetesにおけるログの収集](documents/kubernetes-basics/logging.md) :fa-external-link:
:   Kubernetesクラスター上で動作するコンテナのログを、確認、収集する手順を体験します。

#### サービスメッシュ Istio

- [Istioのインストール](documents/istio/install-istio.md) :fa-external-link:
:   Kubernetesクラスターにサービスメッシュ"Istio"をインストールします。

- [リクエスト・ルーティングの制御](documents/istio/configuring-request-routing.md) :fa-external-link:
:   Istioを使ってリクエスト・ルーティングの制御を体験します。カナリーデプロイメントのシナリオを想定して、複数バージョンが混在するアプリケーションのコンテナ群に対して、バージョン毎に流量を変えてルーティングしてみます。

Appendix.
---------

#### ローカルKubernetesクラスターの作成

- [Vagrant CoreOS Cluster](documents/create-local-cluster/vagrant-coreos-cluster.md) :fa-external-link:
:   Vagrantのインストールスクリプトを利用してKubernetesクラスターを構成します。 この手順では、ローカルPC上に複数のVirtualBox仮想マシンを立ち上げ、それらの上にKubernetesのマルチノードクラスターを構成します。

- [minikube](documents/create-local-cluster/minikube.md) :fa-external-link:
:   minikubeを利用してKubernetesクラスターを構成します。ローカルPCに立ち上げた仮想マシン上に、Kubernetesのシングルノードクラスターを構成することができます。Vagrant CoreOS Clusterに比べて簡単な手順でクラスターを構成できます。
