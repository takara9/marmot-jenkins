# vagrant-jenkins

## 概要

Jenkinsのジョブの中で以下を実行する。

* GitHubからソースコードのクローン
* Dockerコンテナのビルド
* DockerHubレジストリへプッシュ
* Kubernetesクラスタへデプロイ


## Jenkinsの環境

Jenkinsのサーバーは３台構成

* Jenkins マスター  master
* Jenkins エージェント agent1, agent2


Vagrantから上記３台のサーバーを起動して、Ansible playbookにより以下を実行する。

* マスター上でssh鍵の生成
* sshリモート操作環境の設定,hosts,公開鍵配布
* 全ノードへのDocker, kubectl, Java8 のインストール
* マスターに Jenkins と Apache2 をインストールして起動



## ブラウザアクセスするパソコンの設定

パソコンの/etc/hostsに以下のエントリーを加える。

~~~
192.168.1.85 jenkins.labs.local
~~~


## 起動方法

~~~
$ git clone https://github.com/takara9/vagrant-jenkins
$ vagrant up
~~~


## 初期セットアップ

ブラウザから http://jenkins.labo.local をアクセスする。パスワードが書き込まれたファイルのパスと初期パスワード入力の画面が表示される。

`vagrant ssh master`でログインしてパスのファイルを開いてパススワードをコピーして、ブラウザ画面の入力フィールドへペーストしてエンターする。

プラグインの自動インストールか、手動で選択してインストールかの選択が出るので、自動インストールを選択する。

ユーザー名などを設定する画面が現れるので、画面の指示に従ってインプットする。


## ノードの追加

ジョブを実行するためのノードを２台登録する。

* 「Jenkinsの管理」をクリック
* 「ノードの管理」をクリック
* 「新規ノード」の作成をクリック
* ノード名をインプットして、Permanent Agent にマークして、OKをクリックする。
* ノード登録の項目を埋めて保存ボタンをクリックする。
    * ノード名 表示のためのフィールド
    * 同時ビルド数は、vCPUの数と合わせ 1 とする
    * リモートFSルートは、/var/jenkins をインプット
    * ラベル Linux
    * 用途 「このスレーブをできるだけ利用する」を選択
    * 起動方法 「SSH経由でUnixマシンのスレーブエージェントを起動」を選択
    * ホスト名、ユーザー名(root)、秘密鍵を設定する。秘密鍵は/vagrant/keysに生成されたid_rsaを指定する。
    * "Host Key Verification Strategy" は、"Non verifying Verification Strategy"を選択
    * 他の項目はデフォルトを利用

* 保存をクリックした後、リストから追加したノードをクリックして、ReLunch Agentをクリックしてエージェントを起動する。


## Jenkinsプラグインを導入

プラグインは、以下を選択する。

* Blue Ocean

「Jenkinsの管理」->「プラグインの管理」->「利用可能」をクリックする。そして、filterにBlueと入れて絞り込まれたリストから、対象にチェックをいれる。「ダウンロード後再起動」を選択して、Jenkinsの再起動が完了するのを待つ。






