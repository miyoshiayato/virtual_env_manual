# 環境構築手順書  
---
## 目次
1. バージョン一覧
2. 環境構築
3. 環境構築の所感
4. 参考サイト  
<br>
<br>

### 1.バージョン一覧
* PHP 7.3
* Nginx 1.21.0
* MySQL 5.7
* Laravel 6.0
* CentOS7  
<br>
<br>

### 2.環境構築
#### Vagrantの作業ディレクトリを用意する  
Vagrantの作業用ディレクトリを作成します。  
以下のいずれかのディレクトリの下に manual_test という名前でディレクトリを作成します。  
* 自分の作業ディレクトリ
* デスクトップ  

```
mkdir manual_test
```
作成したフォルダの中で以下のコマンドを実行します。  
```
cd manual_test
# vagrant init box名 先ほどダウンロードしたboxを使用することになります
vagrant init centos/7

# 実行後問題なければ以下のような文言が表示されます
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```
<br>

#### Vagrantfileの編集
今回行う編集は、三箇所です。

```
# 変更点①
config.vm.network "forwarded_port", guest: 80, host: 8080

# 変更点②
config.vm.network "private_network", ip: "192.168.33.19"
```
上記二箇所の # が付いているのを外します。
また以下の箇所はコメントインし、変更を加えてください。
```
# 変更点③
config.vm.synced_folder "../data", "/vagrant_data"
# ↓ 以下に編集
config.vm.synced_folder "./", "/vagrant", type:"virtualbox"
```
`./` はカレントディレクトリ(vagrant_test)を示しており、ホストOS (Mac or Windows) のvagrant_testディレクトリ内とゲストOS (Vagrant) の `/vagrant` のディレクトリ内をリアルタイムで同期するための設定です。
<br>

#### Vagrant プラグインのインストール
Vagrantには様々なプラグイン(拡張機能)が用意されています。今回は`vagrant-vbguest`というプラグインをインストールします。
vagrant-vbguestは初めに追加したBoxの中にインストールされているGuest Additionsというもののバージョンを、VirtualBoxのバージョンに合わせて最新化してくれるプラグインです。
```
vagrant plugin install vagrant-vbguest
```
vagrant-vbguestのインストールが完了しているか下記のコマンドを実行して確認しましょう。
```
vagrant plugin list
```
<br>

#### Vagrantを使用してゲストOSの起動
以上で仮想環境を構築する準備は整いました。
Vagrantfileがあるディレクトリにて以下のコマンドを実行して、早速起動してみましょう。
```
vagrant up
```
初回の起動には、時間がかかりますので気長に待ちましょう。

※ `vagrant up`すると、以下のようにエラーメッセージが表示されマウントできない場合がございます。
```
・・・中略・・・

No package kernel-devel-3.10.0-1127.el7.x86_64 available.
Error: Nothing to do
Unmounting Virtualbox Guest Additions ISO from: /mnt
umount: /mnt: not mounted
==> default: Checking for guest additions in VM...
    default: No guest additions were detected on the base box for this VM! Guest
    default: additions are required for forwarded ports, shared folders, host only
    default: networking, and more. If SSH fails on this machine, please install
    default: the guest additions and repackage the box to continue.
    default:
    default: This is not an error message; everything may continue to work properly,
    default: in which case you may ignore this message.
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!

umount /mnt

Stdout from the command:



Stderr from the command:

umount: /mnt: not mounted
```
<u>原因</u>
vboxの内容が CentOS 7.8で構成されているが、CentOS 7.9がリリースされてしまったため、インストールされているカーネルのバージョンと、一致する kernel-devel パッケージがリポジトリから取得できなくなってしまった。

vbguest を使用しているため VirtualBox Guest Additions の更新で 該当するバージョンの kernel-devel を取得できずに失敗している。

<u>対処方法</u>
`vagrant up`で失敗しても、SSH接続できるところまで起動できているので、次の手順で一度 kernel をアップデートし、完了させる。

1. `vagrant up` で失敗
2. `vagrant ssh`
3. `sudo yum -y update kernel`
4. `exit`
5. `vagrant reload --provision`

あくまでも応急処置なので、自分でアップデート済みのvboxを作成するか、公式のvboxがアップデートするのを待つしかありません。
<br>

#### ゲストOSへのログイン
起動が正常に終了すれば、皆さんが使用しているPCのOSであるホストOSの上に全く別のゲストOSが立ち上がったことになります。
では実際にゲストOSにログインしていきましょう。
*ssh*
`ssh` はリモートマシンにログインするコマンドです。
今回は、VirtualBoxが提供するネットワークを通じてターミナル上でホストOSからゲストOS(リモートマシン)にログインします。

作成した `manual_test` ディレクトリに移動して下記のコマンドを実行しましょう。
```
$ vagrant ssh
```
コマンドを実行した後、以下のような表記になっていればゲストOSにログインしていることになります。
```
Welcome to your Vagrant-built virtual machine.
[vagrant@localhost ~]$
```
<br>

#### Vagrant 仮想環境構築
構築したゲストOSには、まだ開発するにあたって必要となるソフトウェアやコマンドなどがインストールされていない状態です。
まずはパッケージというものをインストールして開発を進められる環境を整えていきます。
主に以下の3点をインストールしていきます。
* パッケージのインストール
* PHPのインストール
* composerのインストール

パッケージの導入にあたって実行するコマンドが以下になります。
```
sudo yum -y install パッケージ名
sudo yum -y groupinstall "導入する名称"
```
<br>

##### パッケージのインストール
では早速グループパッケージをインストールしていきましょう。
下記のコマンドを実行してください。
```
sudo yum -y groupinstall "development tools"
```
このコマンドを実行することによりgitなどの開発に必要なパッケージを一括でインストールできます。
<br>

##### PHPのインストール
次はPHPをインストールしていきます。

yumコマンドを使用してPHPをインストールした場合、古いバージョンのPHPがインストールされてしまいます。
Laravelを動作させるにはPHPのバージョン7以上をインストールする必要があるため
yumではなく外部パッケージツールをダウンロードして、そこからPHPをインストールしていきます。
今回は、PHPバージョン7.3をインストールします。
```
sudo yum -y install epel-release wget
sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
sudo rpm -Uvh remi-release-7.rpm
sudo yum -y install --enablerepo=remi-php73 php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip
php -v
```
PHPのバージョンを確認できましたでしょうか？ PHPのインストールは、以上で終わりです。
<br>

##### composerのインストール
次にPHPのパッケージ管理ツールであるcomposerをインストールしていきます。
```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"

# どのディレクトリにいてもcomposerコマンドを使用できるようfileの移動を行います
sudo mv composer.phar /usr/local/bin/composer
composer -v
```
composer のバージョンが確認できたらOKです。

以上の作業でゲストOS内にPHPとcomposerコマンドの実行環境が整いました。
<br>

#### Vagrant 仮想環境構築(2)
先程は、ゲストOS内にパッケージ、PHP、Composerの3点をインストールしました。
今回はWebサーバとデータベースをインストールして、最終的にLaravelアプリケーションを動かします。
##### Laravelアプリケーションの作成
アプリケーションをゲストOS内で動かしていくにあたって、Laravelアプリケーションを作成していきます。

まず現在ゲストOSにログインしている人は一旦`exit`コマンドを実行してログアウトしてください。
その後、以下のコマンドを自身の環境に合わせて実行し、composerを使用してLaravel6.0をmanual_testディレクトリ下に作成します。
```
$ cd manual_test
$ composer create-project laravel/laravel=6.0 laravel_project
```
それでは、Laravelがしっかりとインストールできたか確認しておきましょう！

作成したLaravelプロジェクトに移動してください。
```
$ cd laravel_project/
```
次のコマンドでLaravelのバージョンが表示されますので、しっかりインストールされていることの確認ができました。
```
$ php artisan --version
Laravel Framework 6.20.29
```
<br>

##### Nginxのインストール
次に、WebサーバソフトウェアであるNginxをインストールしていきます。

まだゲストOSにログインしていない方は`manual_test`ディレクトリにて`vagrant ssh`コマンドを実行してログインしてください。
Nginxの最新版をインストールしていきます。
viエディタを使用して以下のファイルを作成します。
```
$ sudo vi /etc/yum.repos.d/nginx.repo
```
書き込む内容は以下になります。
```
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=0
enabled=1
```
書き終えたら保存して、以下のコマンドを実行しNginxのインストールを実行します。
```
$ sudo yum install -y nginx
$ nginx -v
```
Nginxのバージョンが表示されますので、インストールされていることの確認ができました。
ではNginxの起動をしましょう。
```
$ sudo systemctl start nginx
```
ブラウザにて http://192.168.33.19 と入力し、NginxのWelcomeページが表示されましたでしょうか？
表示されたら問題なく動いていますので次に進みましょう。
<br>

##### Laravelを動かす
Nginxには、設定ファイルが存在しているので編集を行います。
また、Nginxは、php-fpmとセットで使用ます。。
php-fpmにも設定ファイルが存在しているのでこちらも編集を行います。

では、早速Nginxの設定ファイルを編集していきます。
使用しているOSがCentOSのため、`/etc/nginx/conf.d` ディレクトリ下の `default.conf` ファイルが設定ファイルとなります。
```
$ sudo vi /etc/nginx/conf.d/default.conf
```
編集範囲がやや広いので気をつけましょう。
コメント外しのミスが多いのでしっかりと確認してください。
```
server {
  listen       80;
  server_name  192.168.33.19; # Vagranfileでコメントを外した箇所のipアドレスを記述してください。
  # ApacheのDocumentRootにあたります
  root /vagrant/laravel_project/public; # 追記
  index  index.html index.htm index.php; # 追記

  #charset koi8-r;
  #access_log  /var/log/nginx/host.access.log  main;

  location / {
      #root   /usr/share/nginx/html; # コメントアウト
      #index  index.html index.htm;  # コメントアウト
      try_files $uri $uri/ /index.php$is_args$args;  # 追記
  }

  # 省略

  # 該当箇所のコメントを解除し、必要な箇所には変更を加える
  # 下記は root を除いたlocation { } までのコメントが解除されていることを確認してください。

  location ~ \.php$ {
  #    root           html;
      fastcgi_pass   127.0.0.1:9000;
      fastcgi_index  index.php;
      fastcgi_param  SCRIPT_FILENAME  /$document_root/$fastcgi_script_name;  # $fastcgi_script_name以前を /$document_root/に変更
      include        fastcgi_params;
  }

  # 省略
```
Nginxの設定ファイルの変更は、以上です。
次に php-fpm の設定ファイルを編集していきます。
```
$ sudo vi /etc/php-fpm.d/www.conf
```
変更箇所は以下になります。
```
;24行目近辺
user = apache
# ↓ 以下に編集
user = nginx

group = apache
# ↓ 以下に編集
group = nginx
```
設定ファイルの変更に関しては、以上となります。
では早速起動しましょう(Nginxは再起動になります)。

```
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```

再度ブラウザにて、 http://192.168.33.19 を入力して確認してください。
Laravelアプリケーションを動かすことはできたでしょうか？？
今のままだた画面すら表示されずの状態かと思います。
ではなぜこのような画面が表示されているのか、答えは、 ブラウザに書いてあります。

表示されている文言の下部に「ファイヤーウォール」という単語が確認できますでしょうか？
聞きなれない単語かと思いますが、セキュリティの観点では「ファイヤーウォール」は大切な機能であるため、起動状態のままホストOS側からアクセスできるようにしてあげましょう。

Vagrantfileの編集をした際を思い出してください。
config.vm.network "forwarded_port", guest: 80, host: 8080 と記述されていた箇所のコメントを解除したと思います。
この `80` という数字は、httpという通信を行うためのポートと呼ばれる窓口番号です。
なのでファイヤーウォールに対してこの80ポートを経由したhttp通信によるアクセスを許可するためのコマンドを実行します。
```
# ファイヤーウォールの起動
$ sudo systemctl start firewalld.service
$ sudo firewall-cmd --add-service=http --zone=public --permanent

# 新たに追加を行ったのでそれをファイヤーウォールに反映させるコマンドも合わせて実行します
$ sudo firewall-cmd --reload
```
では、一旦この状態で画面を確認してみます。

もしまだ表示できないようであれば、一度以下のコマンドを実行してください。
```
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```
Laravelのwelcome画面が表示されたでしょうか？
恐らく、まだ表示されず、エラーが出るかと思います。
Forbidden 403というエラーでなければ、
>ログファイルへの書き込みができる権限を付与

こちらへ進んでください。
<br>

##### Forbidden 403というエラーが表示された場合
Laravelのwelcomeページが表示されず、 Forbidden 403 というエラーが出た場合の対処法を以下に記載します。
(Laravelのwelcomeページが表示された場合は読み飛ばしてしまって問題ありません。)

viエディタを使用してSELinuxの設定を変更します。
「SELinux コンテキスト」の不一致によりエラーが出ているので、SELinuxを無効化します。(今回はローカル環境なので無効にしてしまっても問題ありませんが、本番環境構築のときは別のアプローチが必要です。)
```
$ sudo vi /etc/selinux/config
```
viエディタが開き設定ファイルが表示されるので下記の部分を探してください。
```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
# enforcing - SELinux security policy is enforced.
# permissive - SELinux prints warnings instead of enforcing.
# disabled - No SELinux policy is loaded.
SELINUX=enforcing
```
こちらの記述を下記のように書き換えて、保存してください。

`SELINUX=disabled`

設定を反映させるためにゲストOSを再起動する必要があるので、ゲストOSをから一度ログアウトして下記コマンドを実行してください。
```
$ exit
$ vagrant reload
```
リロードが完了したら再度ゲストOSにログインしましょう。
```
$ vagrant ssh
```
再度Nginxを起動します。
```
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```
<br>

##### ログファイルへの書き込みができる権限を付与
上記設定も終わり、ブラウザにて、 http://192.168.33.19 を入力して確認してください。
ようやく画面が表示される！...と思いきや、以下のようなLaravelのエラーが表示されると思います。
```
The stream or file "/vagrant/laravel_project/storage/logs/laravel.log" could not be opened: failed to open stream: Permission denied
```
これは 先程php-fpmの設定ファイルの user と group を `nginx` に変更したと思いますが、ファイルとディレクトリの実行 user と group に `nginx` が許可されていないため起きているエラーです。

では、以下のコマンドを実行して `nginx` というユーザーでもログファイルへの書き込みができる権限を付与してあげましょう。
```
$ cd /vagrant/laravel_project
$ sudo chmod -R 777 storage
```
これで、http://192.168.33.19 にアクセスすると正常にLaravelのWelcome画面が表示されると思います。
<br>

##### データベースのインストール
今回インストールするデータベースはMySQLとなります。versionは5.7を使用します。

centos7は、デフォルトでmariaDBというデータベースがインストールされていますが、MariaDBはMySQLと互換性があるDBなので気にせず、MySQLのインストールを進めていきます。

rpmに新たにリポジトリを追加し、インストールを行います。
```
sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
sudo yum install -y mysql-community-server
mysql --version
```
versionの確認ができましたらインストール完了です。
次にMySQLを起動し接続を行います。
```
sudo systemctl start mysqld
mysql -u root -p
Enter password:
```
MacもしくはWindowsにMySQLをインストールしたときは、何も入力せずに接続が可能だったかと思いますが今回はデフォルトでrootにパスワードが設定されてしまっています。
まずはpasswordを調べ、接続しpassswordの再設定を行っていく必要があります。

※ 今回は、比較的簡単な方法でパスワードの再設定を行いますが、セキュリティ的によろしくはないため本番環境と呼ばれる環境でこの方法で再設定するのは避けてください。
```
sudo cat /var/log/mysqld.log | grep 'temporary password'  # このコマンドを実行したら下記のように表示されたらOKです
2017-01-01T00:00:00.000000Z 1 [Note] A temporary password is generated for root@localhost: hogehoge
```
`hogehoge` と記載されている箇所に存在するランダムな文字列がパスワードとなります。

出力したランダム文字列をコピー後、再度以下のコマンドを実行し、パスワード入力時にペーストしてください。
```
$ mysql -u root -p
$ Enter password:
mysql >
```
問題なく接続できたでしょうか？
次に接続した状態でpasswordの変更を行います。
```
mysql > set password = "新たなpassword";
```
<br>
新たなpasswordには、必ず大文字小文字の英数字 + 記号かつ8文字以上の設定をする必要があります。
MySQL5.7のパスワードポリシーは厳格で開発段階では非常に面倒のため、以下の設定を行いシンプルなパスワードに初期設定できるようにMySQLの設定ファイルを変更します。(ステージング環境や本番環境で使用するDBにはポリシーを遵守したパスワードを設定してください。)
```
$ sudo vi /etc/my.cnf
```

```
# 省略

[mysqld]

# 省略

# read_rnd_buffer_size = 2M
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# 下記の一行を追加
validate-password=OFF
```
編集後はMySQLサーバの再起動が必要です。
```
$ sudo systemctl restart mysqld
```
再度MySQLにログインしてパスワードの初期設定を行えば、簡単なパスワードで登録ができます。
以上で、MySQLの導入と設定が完了となります。
<br>

##### データベースの作成
実際にLaravelアプリケーションを動かす上で使用するデータベースの作成を行います。
```
mysql > create database laravel_project;
```
Query OKと表示されたら作成は完了となります。
下記コマンドにて終了しましょう。
```
mysql > exit
```
<br>

##### Laravelアプリケーションログイン機能の作成
作成したデータベースに接続するため、laravel_projectディレクトリ下の .env ファイルの内容を以下に変更してください。
```
$ vi /vagrant/laravel_project/.env
```
```
DB_PASSWORD=
# ↓ 以下に編集
DB_PASSWORD=登録したパスワード
```
次に、ログイン機能の実装を行います。

>Laravel5.xでは「php artisan make:auth」コマンドで簡単にLOGIN機能を作成できていました。
※Laravel6.x以降 php artisann make:auth コマンドは無くなりました。

1. laravel/uiをインストール
```
#Laravel6.x 
composer require laravel/ui:^1.0 --dev
```
**【要注意】** 
laravel/ui コマンドにバージョンを付け忘れないよう注意してください。
※[Laravel6.x 公式](https://laravel.com/docs/6.x/frontend#introduction)

2. LOGIN機能＆テーブル作成
```
php artisan ui vue --auth

php artisan migrate
```
これでLaravel6のLOGIN機能が実装できました！
<br>
<br>

### 3.環境構築の所感
ローカルPCに個人開発環境を建てたいという時に、`vagrant(virtualbox)`、`docker`の方法を学ぶ事ができました。
また、WEBサーバソフトウェアも`Apache`と`Nginx`がありそれぞれにメリット・デメリットがあります。
これから仮想環境の構築をする際は、メリットデメリットの把握を行い、どう使い分けていくか考える事が大事だと感じました。
また、環境構築中、PCがシャットダウンしてしまう事が多々ありました。対処法としてアクティビティモニタでのメモリの確認や電源ボタンからエネルギーを著しく消費中のアプリを確認しておりました。
Webページやターミナルのタブの開きすぎは、メモリの使用に繋がるので、意識すべきだと学びました。
<br>
<br>

### 4.参考サイト
https://qiita.com/daisu_yamazaki/items/a914a16ca1640334d7a5

https://qiita.com/mao172/items/f1af5bedd0e9536169ae

https://atmarksharp.v01.jp/posts/markdown-cheat-sheet.html

https://weblabo.oscasierra.net/beginning-vagrant-sahara-1/



