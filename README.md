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

---

## メモ

* harbor, gitlab のCA証明書を /etc/ssl/certs へ配置しなければならない。
* jenkins は docker のグループに入る必要がある。
* Harbor のプライベートリポジトリの認証情報を登録する必要がある。


## Jenkinsパイプラインの自動実行

Jenkinsから変更をポーリングして変更を検知してジョブを実行する方法と
git push を実行したタイミングでビルドのジョブが走る方法の二つの設定を記述する。
これらの設定をしなくても、Jenkinsの管理画面からパイプラインのジョブを実行できる。

### 自動実行のJenkins側の前提条件は以下
* Harbor が起動していること
* Harbor の CA証明書が JenkinsサーバーのLinuxにインポートされていること
* GitLab が起動していること
* GitLan の CA証明書が JenkinsサーバーのLinixにインポートされていること
* JenkinsにGitLab Plugin がインストールされていること
* Jenkins設定でGitLabのパラメーターが設定されていること。
   * Connection name
   * Gitlab host URL
   * Credentials
   * Test Connection が成功すること

### GitLab側の前提条件は以下
* rootでログインして、スパナマークのアイコン(Admin Area) の Setting -> Network -> Outbound requests -> Allow requests to the local network from web hooks and services にチェックが入っていること
* Jenkingと連携するユーザーでログインして、Settings -> Webhooks に設定がされていること。
   1. URL に Jenkins プロジェクトのURL
   1. 秘密トークンに、Jenkins のプロジェクト、ビルドトリガーの高度な設定で生成したSercret Tokenがセットされていること。
   1. SSL証明書検証の有効化のチェックが外されていること（無効になっていること）
   1. テストボタンをクリックして、連携が成功すること


### GitLabリポジトリを5分間隔でポーリングして変更があった場合にビルドを実行する

1.「新規ジョブ作成」をクリック
1. ジョブの名前（日本語でもOK)をインプット
1. 全体のタブ
    1. GitLab Connection にコネクション名をセット（事前登録必要）
    1. GitLab Repository Name にリポジトリ名をセット
1. ビルドトリガのタブをクリック、
    1. SCMをポーリングにチェック
    1. "H/5 * * * *" をインプット 毎時*5分でポーリング
1. パイプラインのタブをクリック
    1. Pipeline script from SCMを選択
    1. SCM: Git を選択
        1. リポジトリURL: https://gitlab.labo.local/tkr/webapl-2.git
        1. 認証情報: https://gitlab.labo.local/　のユーザーID/パスワードを追加して選択
        1. ビルドするブランチ "*/master" -> "*/main"へ変更
    1. Script Path:  Jenkinsfile
1. 「保存」をクリック



### GitLabへpushされた時に、ビルドを実行する設定

1.「新規ジョブ作成」をクリック
1. ジョブの名前（日本語でもOK)をインプット
1. 全体のタブ
    1. GitLab Connection にコネクション名をセット（事前登録必要）
    1. GitLab Repository Name にリポジトリ名をセット
1. ビルドトリガのタブをクリック
    1. Build when a change is pushed to GitLab. GitLab webhook にチェック
    1. Enabled GitLab triggersのPush Eventsにチェック
    1. Rebuild open Merge Requests で On push to source brance を選択
    1. 高度な設定をクリック
        1. Secret token をクリック
        1. Generate クリックして、トークンを生成
1. パイプラインのタブをクリック
    1. Pipeline script from SCMを選択
    1. SCM: Git を選択
        1. リポジトリURL: https://gitlab.labo.local/tkr/webapl-2.git
        1. 認証情報: https://gitlab.labo.local/　のユーザーID/パスワードを追加して選択
        1. ビルドするブランチ "*/master" -> "*/main"へ変更
    1. Script Path:  Jenkinsfile
1. 「保存」をクリック



