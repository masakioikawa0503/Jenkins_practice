# Jenkins_practice
Jenkins(AWS)とGithubの連携を行うにあたり、備忘録として手順下記に記述

～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～

# 目的
・ローカル環境からGithubへPush通知を行うたびに、Push通知を受け取ったGithubはJenkinsにwebhookを行いビルドを実施させる環境を構築する
<br>
# 検証環境
・ローカル環境：Windows Subsystem for Linux 2 (WSL2 on Ubuntu)　※cat /etc/os-release

<br>
・Github:
    URL→https://github.com/masakioikawa0503/jenkins
    リポジトリ→https://github.com/masakioikawa0503/jenkins.git
<br>
・Jenkins
    EC2 on AWSで実装
    (簡単に検証環境を構築したいため、「Terraterm」でVPC、パブリックサブネット、IGW、セキュリティグループ、EC2を一気に作成する）
<br>
# 検証条件
・本当はJenkinsサーバ（CI）とデプロイ先サーバ(CD)を分けての検証を視野に入れたが以下の理由により今回は1台（CIサーバのみ）として実装
    EC2は従量課金で2台だと単純に2倍コストがかかるため
    GithubとJenkinsの連携部分を初めて構築するため、まずは簡単なところから確実に実装して検証していきたいから。
    GithubとJenkinsの連携の内容は以下とする
    javaの簡単なサンプルプログラミングをローカルからGithubへpush
        Githubはpush通知があったので、Jenkinsに通知する
        Jenkinsはpush通知を受け取ったGithubの該当リポジトリを参照してファイルを取得して簡単なテスト（コンパイル、java実行）を実行する
        (Jenkinsでは事前にGithubの該当リモートリポジトリからgit cloneをしてリポジトリを取得する)

# 手順
・以下に私が実施した手順(例)を示す
（手順は主にGithubとJenkinsの連携部分に焦点をあてて記述していく。例えばGithubのリポジトリの作り方等は他サイトを参照して作成する）



#Github
=========================
①検証用リポジトリを作成
https://github.com/masakioikawa0503/jenkins.git

②サンプルファイルを作成する
作成ファイルは上記リポジトリを参照

=========================

↓
↓
↓

#ローカル(WSL2)
=========================
③git clone https://github.com/masakioikawa0503/jenkins.git
→検証用リポジトリがダウンロードされたか確認する

④EC2環境をterraformで構築する
下記をgitクローン後Terraform_AWSディレクトリに進み、「terraform init→terraform plan→terraform apply」を実行する 
https://github.com/masakioikawa0503/Terraform_AWS.git

　詳細なterraformの使い方は公式ドキュメントを参考
　https://registry.terraform.io/providers/hashicorp/aws/latest/docs
=========================

↓
↓
↓

#Jenkins（AWS）
=========================
⑤WSL2からsshでterraformで構築したEC2にアクセスし、以下参考サイトを基にJenkinsを実装

また、JenkinsをEC2に導入する方法は以下の参考サイトを参考に導入
https://qiita.com/tamorieeeen/items/15d90adeebbf8b408c78
↓
（以下、参考サイトより）
1.JDK 8のインストールを実施する
$ sudo yum update -y
$ sudo yum install -y java-1.8.0-openjdk-devel.x86_64
$ sudo alternatives --config java
$ java -version

2.Jenkinsのyumリポジトリを追加する
$ sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat/jenkins.repo
$ sudo rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key

接続先のbaseurlがhttpだとinstallでコケるのでhttpsに変更する
$ sudo vi /etc/yum.repos.d/jenkins.repo

[jenkins]
name=Jenkins
baseurl=https://pkg.jenkins.io/redhat
gpgcheck=1

3.Jenkinsをインストール
$ sudo yum install -y jenkins
$ rpm -qa | grep jenkins
jenkins-2.202-1.1.noarch

4.Jenkinsを起動
$ sudo systemctl start jenkins
Starting jenkins :                          [  OK  ]

自動起動設定
$ sudo systemctl enable jenkins

パッケージの詳細情報が確認できる
yum info jenkins

5.Jenkinsの設定

初期設定
http://(public IP address):8080にアクセスして画面に従ってまず初期設定する

$ sudo less /var/lib/jenkins/secrets/initialAdminPassword
Unlock Jenkinsは初期パスワードを確認して入力する
Customize JenkinsはとりあえずInstall suggested pulginsを選択
Create First Admin Userで管理者ユーザーを登録
Instance Configurationでhttp://(public IP address):8081/jenkins/と入力
Save and Finishで設定完了
URLとportの設定を変更する
初期設定の4に書いたようにhttp://(public IP address):8081/jenkins/で動かしたいので、設定を変更してJenkinsを再起動する

$ sudo vi /etc/sysconfig/jenkins
JENKINS_PORT="8081"
JENKINS_ARGS="--prefix=/jenkins"

$ sudo systemctl restart jenkins
http://(public IP address):8081/jenkins/にアクセスして初期設定した管理者ユーザーでログインできれば完成


⑥Jenkinsからgitを使えるようにgitをインストールする
yum install -y git


⑦Jenkins（http://(public IP address):8081/jenkins/）にアクセスして設定する
ダッシュボードで「新規ジョブ作成」
　→Enter an item nameに適当な名前を付けて、「フリースタイル・プロジェクトのビルド」を選択し、ＯＫをクリック

以下設定項目
●General
「Github project」にレ点を付けて、該当レポジトリのあるURLを記載
例：https://github.com/masakioikawa0503/jenkins/

●ソースコード管理
「Git」を選択
リポジトリURLは、~.gitを指定（例：https://github.com/masakioikawa0503/jenkins.git）
認証情報は今回検証用なので「なし」のまま
ビルドするブランチもデフォルト「*/master」のまま
リポジトリブラウザもデフォルト「自動」

●ビルド・トリガ
GitHub hook trigger for GITScm pollingにレ点を入れる

●ビルド環境
設定なし（レ点チェック不要）

●ビルド
「ビルド手順の追加」をクリックし「シェルの実行」を選択
「シェルの実行」欄に以下を記載（冒頭に述べたjavaサンプルプログラムのコンパイルと実行）

[シェルスクリプト]
cd /var/lib/jenkins/jenkins
git pull https://github.com/masakioikawa0503/jenkins.git
javac Hello.java
java Hello


上記投入確認後、保存をクリック

⑧Jenkins→Githubにアクセスする際に公開鍵の作成
以下サイトを参考に実施
https://qiita.com/pakiran/items/458fae106566c6c3c963
（以下、参考サイトより）

1.Jenkins用のGithubアカウントを作成
Jenkinsがリポジトリからソースコードを取得してきたり、テスト結果をGitHubに通知するためのJenkins専用GitHubアカウントを作成します。

注：このとき公開鍵認証に必要なSSH keyのパスフレーズは空で設定しましょう！


2.Jenkinsユーザの設定
vimでfalseをbashに変更する

$ cat /etc/passwd/
jenkins:x:220:499:Jenkins Continuous Build server:/var/lib/jenkins:/bin/false
$ sudo vim /etc/passwd
jenkins:x:220:499:Jenkins Continuous Build server:/var/lib/jenkins:/bin/bash

jenkinsフォルダの権限を変更
$ ls -la /var/lib/jenkins
$ chown -R jenkins:jenkins /var/lib/jenkins

Jenkinsユーザでログイン
$ sudo su jenkins
jenkinsユーザでログイン後はbash-4.1$と表示されるようになります。

3.公開鍵を作成(今回パスフレーズ等は空で作成)
bash-4.1$ cd /var/lib/jenkins/.ssh/
bash-4.1$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/jenkins/.ssh/id_rsa):（そのままEnter）
Enter passphrase (empty for no passphrase):（そのままEnter）
Enter same passphrase again:（そのままEnter）
Your identification has been saved in /var/lib/jenkins/.ssh/id_rsa.
Your public key has been saved in /var/lib/jenkins/.ssh/id_rsa.pub.
The key fingerprint is:
be:0d:a3:a8:18:3d:3c:c6:1c:89:5e:1b:a2:b2:2a:aX jenkins@ip-XX-XX-XX-XXX
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
+-----------------+
作成した公開鍵(id_rsa.pub)を、Jenkins用GitHubアカウントに追加（add key）
　→後述


4.jenkinsユーザのgit設定
/var/lib/jankinsディレクトリ下でないとfatal: failed to stat '.': Permission deniedになるので注意！

user.emailとuser.nameにはJenkins用アカウントのemailとuser名を入力

bash-4.1$ git config --global user.email "hoge@XXXXX.co.jp"
bash-4.1$ git config --global user.name "hoge"


以上まで完了後、今回検証用のGithubリモートリポジトリからgit cloneしておく
(参考)
cd /var/lib/jenkins
git clone https://github.com/masakioikawa0503/jenkins.git

上記で器を用意してあげれば、あとはJenkinsのビルドスクリプトで記載したシェルスクリプトで
git push通知を受けるたび、git pullして処理を実施する形になる

=========================

↓
↓
↓

#Github
=========================
⑨右上Settings→「SSH and GPG Keys」を開く
SSH Keysで「New SSH Key」を選択して、Titleとkey(前述した⑧の最後の方で作成したJenkinsの公開鍵の内容をそのままコピ&ペースト)を投入
    →Ass SSH keyをクリック

⑩リモートリポジトリの設定
今回検証用で用意したJenkinsと連携するリポジトリの「Settings」を選択
    →「webhooks」を選択し「Add webhook」をクリック
        →Payload URL：http://[Jenkinsのホスト名（ポート番号やプレフィックスを指定している場合はそれも記載）]/github-webhook/
            (例：〇〇.〇〇.〇〇.〇〇:8081/jenkins/github-webhook/)
    →Content type
        application/x-www-form-urlencordedを選択
    →secret
        記載なし

    →Which events would you like to trigger this webhook?
        Just the push event.を選択

    →Activeにレ点が入っていることをチェック

上記投入後「Update webhook」をクリック
    →適用が成功すると、Webhooksの設定したURLに緑色のレ点が付く（最初は〇印で、適用失敗だと△の警告が表示される）


=========================


■参考サイト

・JenkinsとGithubの連携
https://qiita.com/hika7719/items/4cb50366a4d9fc8415f6
https://qiita.com/pakiran/items/458fae106566c6c3c963

・JenkinsをEC2に導入
https://qiita.com/tamorieeeen/items/15d90adeebbf8b408c78

■参考書籍
Jenkins実践入門
https://www.amazon.co.jp/E6-94-B9-E8-A8-82-E7-AC-AC3-E7-89-88-Jenkins-E5-AE-9F-E8-B7-B5-E5-85-A5-E9-96-80-E2-80-95-E2-80-9/dp/4774189286/ref=dp_ob_title_bk
