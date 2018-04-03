Vagrant CoreOS Cluster
======================
[Kubernetes Vagrant CoreOS Cluster](https://github.com/pires/kubernetes-vagrant-coreos-cluster)にあるインストーラを利用して、Kubernetesクラスターを構成します。
このインストーラーでは、ローカルPC上に複数のVirtualBox仮想マシンを立ち上げ、それらの上にKubernetesマルチノードクラスターを構成します。

併せて、Kubernetesを操作するためのコマンドラインツールである、kubectlのセットアップも行います。

前提条件
--------
この手順は以下の環境で実施することを前提とします。

- OS
    * MacOS X
    * Linux (Ubuntu 16.04)
    * Windows
- インストールSW
    * Vagrant 1.8.6+
    * VirtualBox 5.0.0+

以降の手順は、ほとんどの操作をコマンドラインツールから実行します。__Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell__を利用するものとして、手順を記述します。

!!! Warning
    Linux(ubuntu 16.04)で、VirtualBox 5.1.x以上のバージョンでは動作しない問題を確認しています。当該OSを利用する場合は5.0.xの利用を推奨します。


1. Kubernetesクラスターのセットアップ
-------------------------------------

### 1.1. 準備作業
以下、お使いのマシンの環境に応じて、該当する準備作業を実施してください。

#### :fa-windows: Windowsのみ - gitクライアントの設定変更
このチュートリアルで利用するKubernetesのインストールスクリプトには、Core OSの仮想マシン上で実行されるものが含まれます。

`git clone`時にこれらのスクリプトの改行コードが変換されてしまうと、Core OS上で正しく動作しない場合があります。これを避けるため、改行コードの自動変換を無効化するようにGitを設定しておきます。

以下は、CLIのgitクライアントを利用してこの設定を行う例です。

    git config --global core.autocrlf false; git config --global core.eol lf

#### :fa-apple: MacOS Xのみ - homebrew、wgetのインストール
先の手順でインストールスクリプトを実行すると、Macの場合wgetを使ってkubectl（Kubernetesのコマンドラインツール）がダウンロードされます。このため、事前にwgetをインストールしておく必要があります。

ここではhomebrewを使ってwgetをインストールします。まず、homebrewのインストールです。

    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

次に、以下のコマンドでwgetをインストールします。

    brew install wget

#### :fa-linux: Linuxのみ - NFSのインストール
今回作成するクラスターは、Kubernetesノードの共有ディスクとしてNFSを利用します。NFSがインストールされていなければ、ここでインストールしておきます。

    apt-get install nfs-kernel-server

!!!Note
    WindowsではNFSと同等の機能を提供するVagrantプラグインが自動的に利用されます。Macの場合は標準でNFSがインストールされています。

### 1.2. インストールスクリプトの取得
コマンドラインツールを起動し、適当なディレクトリに[インストールスクリプト](https://github.com/pires/kubernetes-vagrant-coreos-cluster)をクローンします。この時、タグ”1.7.10”を指定します。

    git clone -b 1.7.10 https://github.com/pires/kubernetes-vagrant-coreos-cluster.git

クローンして作成されたディレクトリをカレントディレクトリにしておきます。

    cd kubernetes-vagrant-coreos-cluster

### 1.3. Firewall無効化の確認
NFSへの通信がFirewallによって遮断されてしまうことが多いので、ここで無効化しておきます。利用しているOS/セキュリティソフトウェアの手順に従って操作を行ってください。

!!!Note
    特にWindowsの場合、デフォルトのFirewallの設定でも遮断されてしまうので、注意してください。

### 1.4. インストールスクリプトの実行
インストールスクリプトをデフォルトの設定で実行すると、以下のような構成のクラスターが構築されます。

- マスターノード:
    * ノード数: 1
    * CPU: 2 vCPUs
    * メモリ: 1024 MB
- ワーカーノード:
    * ノード数: 2
    * CPU: 2 vCPUs
    * メモリ: 2048 MB

デフォルトの構成でインストールスクリプトを実行するには、以下のコマンドを実行します。

    vagrant up

利用しているPCが上記のスペックを賄うには非力な場合には、所定の環境変数を設定することによって構成を調整することが可能です。例えば、Macbook Air - Late 2011 4G Memでは、以下の構成でクラスターが稼働することを確認済みです。

- マスターノード:
    * ノード数: 1
    * CPU: 1 vCPUs
    * メモリ: 1024 MB
- ワーカーノード:
    * ノード数: 2
    * CPU: 1 vCPUs
    * メモリ: 1024 MB

この構成でインストールスクリプトを実行するには、以下のコマンドを実行します。

:fa-apple: __Mac__ / :fa-linux: __Linux__

```sh
NODES=2 MASTER_MEM=1024 MASTER_CPUS=1 NODE_MEM=1024 NODE_CPUS=1 vagrant up
```

:fa-windows: __Windows__

```bat
set-item env:NODES -value 2; set-item env:MASTER_MEM -value 1024; set-item env:MASTER_CPUS -value 1; set-item env:NODE_MEM -value 1024; set-item env:NODE_CPUS -value 1; vagrant up
```

!!!Note
    パラメータの詳細は、[インストールスクリプトのREADME.md](https://github.com/pires/kubernetes-vagrant-coreos-cluster/blob/master/README.md)を参照してください。

コンソールに以下のような内容が出力されれば、クラスターの構築が成功しています。

    Bringing machine 'master' up with 'virtualbox' provider...
    Bringing machine 'node-01' up with 'virtualbox' provider...
    Bringing machine 'node-02' up with 'virtualbox' provider...
    ==> master: Running triggers before up...
    ==> master: 2017-11-15 01:25:08 +0900: setting up Kubernetes master...
    ...（中略）
        node-02: Running: inline script
    ==> node-02: Running triggers after up...
    ==> node-02: Waiting for Kubernetes minion [node-02] to become ready...
    ==> node-02: 2017-11-15 01:45:39 +0900: successfully deployed node-02


最後に、以下のコマンドで全ての仮想マシンが稼働していることを確認します。

    vagrant status

正常に稼働している場合、以下のような内容が出力されます。

    Current machine states:
    
    master                    running (virtualbox)
    node-01                   running (virtualbox)
    node-02                   running (virtualbox)

### 1-5. クラスターの停止/再起動
ハンズオン終了後にクラスターを停止/再起動/削除する際には、以下の操作を行ってください

__停止__

    vagrant halt

__起動__

    vagrant up

__削除__（クラスターの停止後に実行）

    vagrant destroy


2. kubectlのセットアップ
------------------------
kubectlはKubernetesの管理操作を行うためのCLIです。ここではkubectlのセットアップをおこないます。

### 2.1. :fa-windows: Windowsのみ - kubectlのインストール
Mac/Linuxの場合は、「2.2. Kubernetesクラスターへの疎通確認」に進んでください。）

!!!Note
    Mac/Linuxの場合は、インストールスクリプトにより自動でkubectlがインストールされます。

kubectlのバイナリファイルを、以下からダウンロードします。

- [kubectl v1.8.0](https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/windows/amd64/kubectl.exe)

ダウンロードしたバイナリを適当なフォルダに配置し、PATHを通します。続いてWindows PowerShellを再起動した後、以下のコマンドを実行してkubectlが動作することを確認します。

    kubectl version

次に、Kubernetesクラスターにアクセスするための設定を行います。インストールスクリプト一式のトップをカレントディレクトリにしておきます。

    cd kubernetes-vagrant-coreos-cluster

以下のコマンドを順に実行していきます。

```bat
kubectl config set-cluster default-cluster --server=https://172.17.8.101 --certificate-authority=%CD%/artifacts/tls/ca.pem
```
```bat
kubectl config set-credentials default-admin --certificate-authority=%CD%/artifacts/tls/ca.pem --client-key=%CD%/artifacts/tls/admin-key.pem --client-certificate=%CD%/artifacts/tls/admin.pem
```
```bat
kubectl config set-context default-cluster --cluster=default-cluster --user=default-admin
```
```bat
kubectl config use-context default-cluster
```

### 2.2. Kubernetesクラスターへの疎通確認
以下のコマンドを実行すると、Kubernetesクラスターの一般的な情報を取得することができます。このコマンドで、クラスターへの疎通を確認します。

    kubectl cluster-info

コンソールに以下のような内容が出力されれば、クラスターとの疎通は問題ありません。

    Kubernetes master is running at https://172.17.8.101
    KubeDNS is running at https://172.17.8.101/api/v1/namespaces/kube-system/services/kube-dns/proxy
    
    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

ここで、ノード一覧の取得も試してみます。

    kubectl get nodes

コンソール出力は以下のようになります。

    NAME           STATUS                     AGE       VERSION
    172.17.8.101   Ready,SchedulingDisabled   12h       v1.7.5
    172.17.8.102   Ready                      12h       v1.7.5
    172.17.8.103   Ready                      12h       v1.7.5


---
以上で、Vagrant CoreOS Clusterの構築、及びkubectlのセットアップは完了です。


参考リンク
----------

kubectl リファレンス(v1.7)
:    [https://v1-7.docs.kubernetes.io/docs/user-guide/kubectl/v1.7/](https://v1-7.docs.kubernetes.io/docs/user-guide/kubectl/v1.7/)
