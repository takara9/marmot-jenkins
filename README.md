# vagrant-jenkins

## 概要

Jenkinsのジョブの中で以下を実行するための環境

* GitHubからソースコードのクローン
* Dockerコンテナのビルド
* DockerHubレジストリへプッシュ
* Kubernetesクラスタへデプロイ

## Jenkinsの環境

master + agent x2 



## 起動方法

~~~
$ git clone https://github.com/takara9/vagrant-jenkins
~~~


## プラグイン

* docker-build-step
* Blue Ocean


