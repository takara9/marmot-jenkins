# marmot-jenkins

## 概要

Jenkinsのジョブの中で以下を実行する。

* GitHubからソースコードのクローン
* Dockerコンテナのビルド
* DockerHubレジストリへプッシュ
* Kubernetesクラスタへデプロイ


## Jenkinsの環境

Jenkinsのサーバーは最大３台構成

* Jenkins マスター  master
* Jenkins エージェント node1, node2


コマンドから上記３台のサーバーを起動して、Ansible playbookにより以下を実行する。

* マスター上でssh鍵の生成
* sshリモート操作環境の設定,hosts,公開鍵配布
* 全ノードへのDocker, kubectl, Java8 のインストール
* マスターに Jenkins と Apache2 をインストールして起動



## ブラウザアクセスするパソコンの設定

パソコンの/etc/hostsに以下のエントリーを加える。
アドレスは一定ではないので、Vagrantfile, Qemukvm.yaml, Qemukvm_s.yamlなどからマスターとなる
JenkinsサーバーのIPアドレスを取得して、/etc/hostsへセットする。

~~~
192.168.1.85 jenkins.labs.local
~~~


## Vagrantでの起動方法

~~~
$ git clone https://github.com/takara9/marmot-jenkins
$ cd marmot-jenkins
$ vagrant up
~~~


## QEMU/KVMでの起動方法

~~~
$ git clone https://github.com/takara9/marmot-jenkins
$ cd marmot-jenkins
$ vm-create -f Qemukvm.yaml
~~~



## 初期セットアップ

ブラウザから http://jenkins.labo.local をアクセスする。パスワードが書き込まれたファイルのパスと初期パスワード入力の画面が表示される。

ログインしてパスのファイルを開いてパススワードをコピーして、ブラウザ画面の入力フィールドへペーストしてエンターする。

プラグインの自動インストールか、手動で選択してインストールかの選択が出るので、自動インストールを選択する。

ユーザー名などを設定する画面が現れるので、画面の指示に従ってインプットする。




## ノードの追加

ジョブを実行するためのノードを２台登録する。

1.「Jenkinsの管理」をクリック
1.「ノードの管理」をクリック
1.「新規ノード」の作成をクリック
1.ノード名をインプットして、Permanent Agent にマークして、OKをクリックする。
1.ノード登録の項目を埋めて保存ボタンをクリックする。
    1.ノード名 表示のためのフィールド
    1.同時ビルド数は、vCPUの数と合わせ 1 とする
    1.リモートFSルートは、/var/jenkins をインプット
    1.ラベル Linux
    1.用途 「このスレーブをできるだけ利用する」を選択
    1.起動方法 「SSH経由でUnixマシンのスレーブエージェントを起動」を選択
    1.ホスト名、ユーザー名(root)、秘密鍵を設定する。秘密鍵はmasterノードにログインして.ssh/id_ras を cat で表示してコピペする。
    1."Host Key Verification Strategy" は、"Non verifying Verification Strategy"を選択
    1.他の項目はデフォルトを利用

1.保存をクリックした後、リストから追加したノードをクリックして、ReLunch Agentをクリックしてエージェントを起動する。



## Jenkinsプラグインを導入

プラグインは、以下を選択する。

1.Folders
1.OWASP Markup Formatter
1.Build Timeout
1.Credntial Binding
1.Timespamper
1.Workspace Cleanup
1.Ant
1.Gradle
1.Pipeline
1.GitHub Branch Source
1.Pipeline: GitHub Groovy Libraries
1.Pipeline: Stage View
1.Git
1.GitLab
1.SSH Build Agents
1.Matrix Authorization Strategy
1.PAM Authentication
1.LDAP
1.Email Extention
1.Mailer
1.Locale

1.Blue Ocean
1.GitLab plugin



「Jenkinsの管理」->「プラグインの管理」->「利用可能」をクリックする。そして、filterにBlueと入れて絞り込まれたリストから、対象にチェックをいれる。同様にGitlabも対象にチェックを入れる。「ダウンロード後再起動」を選択して、Jenkinsの再起動が完了するのを待つ。



## 編集メモ
* jenkins は docker のグループに入る必要がある。
* Harbor のプライベートリポジトリの認証情報を登録する必要がある。


## Jenkinsパイプラインの自動実行

Jenkinsから変更をポーリングして変更を検知してジョブを実行する方法と
git push を実行したタイミングでビルドのジョブが走る方法の二つの設定を記述する。
これらの設定をしなくても、Jenkinsの管理画面からパイプラインのジョブを実行できる。

### 自動実行のJenkins側の前提条件は以下
* Harbor が起動していること
* Harbor の CA証明書が Jenkinsサーバーの/etc/ssl/certs へ配置されていること
* GitLab が起動していること
* GitLan の CA証明書が Jenkinsサーバーの/etc/ssl/certs へ配置されていること
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



