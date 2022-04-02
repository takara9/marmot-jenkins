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

ブラウザから http://jenkins.labo.local をアクセスする。
初期パスワード入力の画面が表示される。

パスワードが書き込まれたファイルのパスが表示されているので、
ログインしてパスのファイルを開いてパスワードをコピーして、
ブラウザ画面の入力フィールドへペーストしてEnterする。


以下どちらかの画面が表示される。本インストール方法では前者の画面が現れるように変わったようです。

* Jenkinsのトップ画面が表示されるので、ユーザー登録とプラグインのインストールを実施する。
* プラグインの自動か手動インストールの選択が出るので、自動インストールを選択する。次に、ユーザー名などを設定する画面が現れるので、画面の指示に従ってインプットする。

後者の画面で進んだ場合は、次の「ユーザー登録」、「Jenkinsプラグイン導入」はスキップする。


## ユーザー登録

Jenkinsの管理 -> ユーザーの管理 -> ユーザー作成　へ進む

管理ユーザーを設定する。


## Jenkinsプラグインを導入

Jenkinsの管理 -> プラグインの管理　へ移動

「利用可能」のタブをクリックする。
filterにBlueと入れて絞り込まれたリストから、対象にチェックをいれる。
同様に他のプラグインもチェックを入れる。
「ダウンロード後再起動」を選択して、Jenkinsの再起動が完了するのを待つ。

--- デフォルトでインストール
1.Folders
1.OWASP Markup Formatter
1.Build Timeout
1.Credential Binding
1.Timespamper
1.Workspace Cleanup
1.Ant
1.Gradle
1.Pipeline
1.GitHub Branch Source
1.Pipeline: GitHub Groovy Libraries
1.Pipeline: Stage View
1.Git
1.SSH Build Agents
1.Matrix Authorization Strategy
1.PAM Authentication
1.LDAP
1.Email Extention
1.Mailer
--- 追加分
1.GitLab
1.Blue Ocean
1.Docker Pipeline





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



harborとgitlabからCAファイルを所得して、/etc/ssl/certsの下へおく




## 編集メモ
* jenkins は docker のグループに入る必要がある。
* Harbor のプライベートリポジトリの認証情報を登録する必要がある。


## Jenkinsパイプラインの自動実行

Jenkinsから変更をポーリングして変更を検知してジョブを実行する方法と
git push を実行したタイミングでビルドのジョブが走る方法の二つの設定を記述する。
これらの設定をしなくても、Jenkinsの管理画面からパイプラインのジョブを実行できる。



### 自動実行のJenkins側の前提条件は以下
* HarborとGitLabが起動していること
* 廃止Ubuntu18.4 では、HarborとGitLabのプライベートCA証明書がJenkinsサーバーの/etc/ssl/certs へ配置されていること
* HarborとGitLabのプライベートCA証明書が/usr/share/ca-certificates に配置され、証明書ファイル名が/etc/ca-certificates.confに追加され、update-ca-certificates が実行され成功していること。

* JenkinsにDocker Pipelineがインストールされていること。
* JenkinsにGitLab Plugin がインストールされていること
* Jenkins管理->システム設定でGitLabのパラメーターが設定されていること。
   * Connection name
   * Gitlab host URL
   * Credentials (アクセストークンの設定は以下の補足）
   * Test Connection が成功すること
   * SSLエラーが出る場合は上記プライベートCA証明書が登録されていることを確認する
   
* Jenkins設定でDeclarative Pipeline (Docker)にパラメーターが設定されていること
   * Docker Label: harbor
   * Docker registry URL:  https://harbor.labo.local
   * Jenkinsfileの中のHarborの認証情報が登録されているか、一致しているか？
   * Registry credentials: ユーザー/パスワード 

* kubeconfigのファイルがアップロードされ、Credentials に登録されていること。
   * Jenkinsの管理 -> Manage Credentials -> Domain(Global)のメニューで Add Credentials を選択
   * 種類でSecret fileを選択
   * Fileの選択でkubectlのkubeconfファイルを選択してアップロードする
   * id は、Kubernetesクラスタが識別できるようにクラスタ名をセットする


### Jenkinsのジョブ作成
* 「ジョブ作成」->ジョブ名入力でインプット
* 「パイプライン」を選択して「OK」をクリック
   ・GitLab Connection: gitlab
   ・ビルドトリガの「Build when a change is pushed to GitLab. GitLab webhook URL: https://jenkins.labo.local/project/webapl-1」にチェック
   ・「高度な設定」をクリック
   ・Secret token: 空欄のままで、[Generate]をクリック
   ・パイプラインの定義: 「Pipeline script from SCM」を選択
   ・SCM: Git を選択
   ・リポジトリURL: https://gitlab.labo.local/tkr/webapl-1
   ・認証情報を選択: （既に登録してあるものを登録するか、追加するか、どちらか）
   ・Script Path: Jenkinsfile をセット
   ・[Apply]をクリックして終了

### GitLabアクセストークンの取得と設定

1. rootをログアウトして、一般ユーザーで入り直す。
1. プロジェクトの一つを選択して、Settings -> Access Tokens へ進む
1. Token name は Jenkins-Webapl-2 などJenkinsクラスタとジョブが判別できるようにインプット
1. Select scopes は すべてにチェックを入れる
1. [Create project access token]をクリック、表示されたトークンをコピー


### GitLab側の前提条件は以下
* rootでログインして、スパナマークのアイコン(Admin Area) の Setting -> Network -> Outbound requests -> Allow requests to the local network from web hooks and services にチェックが入っていること

* Jenkingと連携するユーザーでログインして、当該プロジェクトのSettings -> Webhooks に設定がされていること。
   1. URL に Jenkins プロジェクトのURL
   1. 秘密トークンに、Jenkins のプロジェクト、ビルドトリガーの高度な設定で生成したSercret Tokenがセットされていること。
   1. SSL証明書検証の有効化のチェックが外されていること（無効になっていること）
   1. [Add webhook]をクリック
   1. [テスト]ボタンをクリックして、連携が成功すること

  　ジョブが実行され、GitLabからgit cloneが実行される
  　途中で失敗した時は、該当ジョブの「名前」をクリック、
  　最新の完了ビルドをクリック-> Console Output をクリック
  　エラーメッセージが表示されているので、対処する。
  
  　






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


## トラブル対応

ビルドログで、以下のように、エラーが発生する時は、Jenkinsのサーバーにログインして、
systemctl restart jenkins で再起動することで、解決する。

~~~
+ docker build -t harbor.labo.local/tkr/webapl1:2 .
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post http://%2Fvar%2Frun%2Fdocker.sock/v1.38/build?buildargs=%7B%7D&cachefrom=%5B%5D&cgroupparent=&cpuperiod=0&cpuquota=0&cpusetcpus=&cpusetmems=&cpushares=0&dockerfile=Dockerfile&labels=%7B%7D&memory=0&memswap=0&networkmode=default&rm=1&session=xhqo9gqeju9s1vi9lw31zlt6r&shmsize=0&t=harbor.labo.local%2Ftkr%2Fwebapl1%3A2&target=&ulimits=null&version=1: dial unix /var/run/docker.sock: connect: permission denied
~~~
