バージョン一覧
***
- PHP 7.3
- Nginx 1.19.6
- MySQL 5.7
- Laravel 6.0
- OS 7  

どういう流れで環境構築したか
***

### vagrant box  
[Vagrant Cloud](https://app.vagrantup.com/boxes/search)
```
今回はLinuxのCentOSのバージョン7のbox名 centos/7 を指定して実行してみましょう。  
vagrant box add centos/7  
コマンドを実行すると、下記のような選択肢が表示されます。  
```
```
1) hyperv  
2) libvirt  
3) virtualbox  
4) vmware_desktop  
Enter your choice: 3  
```
今回使用するソフトはVirtualBoxのため、3を選択してenterを押しましょう。 

Vagrantの作業用ディレクトリを作成します。
`mkdir vagrant_test`  
作成したフォルダの中で以下のコマンドを実行します。

`cd vagrant_test`
vagrant init box名 先ほどダウンロードしたboxを使用することになります
`vagrant init centos/7`

実行後問題なければ以下のような文言が表示されます  

    A Vagrantfile has been placed in this directory. You are now  
    ready to vagrant up your first virtual environment! Please read  
    the comments in the Vagrantfile as well as documentation on  
    vagrantup.com for more information on using Vagrant.  

### Vagrantfileの編集  
今回行う編集は、三箇所です。  
```
変更点①  
config.vm.network "forwarded_port", guest: 80, host: 8080  

変更点②  
config.vm.network "private_network", ip: "192.168.33.10"  
上記二箇所の # が付いているのを外します。  
また以下の箇所はコメントインし、変更を加えてください。  

変更点③  
config.vm.synced_folder "../data", "/vagrant_data"  
↓ 以下に編集  
config.vm.synced_folder "./", "/vagrant", type:"virtualbox"  
./ はカレントディレクトリ(vagrant_test)を示しており、ホストOS (Mac or Windows) のvagrant_testディレクトリ内とゲストOS (Vagrant) の /vagrant のディレクトリ内をリアルタイムで同期するための設定です。
```
### Vagrant プラグインのインストール  
Vagrantには様々なプラグイン(拡張機能)が用意されています。今回は vagrant-vbguest というプラグインをインストールします。  
vagrant-vbguestは初めに追加したBoxの中にインストールされているGuest Additionsというもののバージョンを、VirtualBoxのバージョンに合わせて最新化してくれるプラグインです。  

`$ vagrant plugin install vagrant-vbguest`  
vagrant-vbguestのインストールが完了しているか下記のコマンドを実行して確認しましょう。  

`$ vagrant plugin list`  

Vagrantfileがあるディレクトリにて以下のコマンドを実行して、早速起動してみましょう。  

`$ vagrant up`  

作成した vagrant_test ディレクトリに移動して下記のコマンドを実行しましょう。  

`$ vagrant ssh`  
コマンドを実行した後、以下のような表記になっていればゲストOSにログインしていることになります。  

`[vagrant@localhost ~]$`  

`$ sudo yum -y groupinstall "development tools"`  
このコマンドを実行することによりgitなどの開発に必要なパッケージを一括でインストールできます。  

### PHPのインストール  
次はPHPをインストールしていきます。  

yumコマンドを使用してPHPをインストールした場合、古いバージョンのPHPがインストールされてしまいます。  
Laravelを動作させるにはPHPのバージョン7以上をインストールする必要があるため  
yumではなく外部パッケージツールをダウンロードして、そこからPHPをインストールしていきます。  
```
sudo yum -y install epel-release wget  
sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm  
sudo rpm -Uvh remi-release-7.rpm  
sudo yum -y install --enablerepo=remi-php73 php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip  
php -v  
```
### MySQL 
今回インストールするデータベースはMySQLとなります。versionは5.7を使用します。  
```
sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm  
sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm  
sudo yum install -y mysql-community-server  
mysql --version  
```
次にMySQLを起動し接続を行います。  

`sudo systemctl start mysqld`  

今回はデフォルトでrootにパスワードが設定されてしまっています。  
```
sudo cat /var/log/mysqld.log | grep 'temporary password'  # このコマンドを実行したら下記のように表示されたらOKです  
2017-01-01T00:00:00.000000Z 1 [Note] A temporary password is generated for root@localhost: hogehoge  
```

では先程出力したランダム文字列をコピー後、再度以下のコマンドを実行し、パスワード入力時にペーストしてください。  
```
$ mysql -u root -p  
$ Enter password:  
mysql >  
```
問題なく接続できたでしょうか？  
次に接続した状態でpasswordの変更を行います。  

`mysql > set password = "新たなpassword";`  

`sudo vi /etc/my.cnf`  
```
#省略  

[mysqld]  

#省略  

# read_rnd_buffer_size = 2M  
datadir=/var/lib/mysql  
socket=/var/lib/mysql/mysql.sock  

# 下記の一行を追加  
validate-password=OFF  
```
編集後はMySQLサーバの再起動が必要です。  

`$ sudo systemctl restart mysqld`  

laravel_appディレクトリ下の .env ファイルの内容を以下に変更してください。  
```
DB_PASSWORD=  
# ↓ 以下に編集  
DB_PASSWORD=登録したパスワード  
```
laravel_appディレクトリに移動して php artisan migrate を実行します。  

### Nginx
Nginxの最新版をインストールしていきます。  
viエディタを使用して以下のファイルを作成します。  

`$ sudo vi /etc/yum.repos.d/nginx.repo`  
書き込む内容は以下になります。  
```
[nginx]  
name=nginx repo  
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/  
gpgcheck=0  
enabled=1  
```
書き終えたら保存して、以下のコマンドを実行しNginxのインストールを実行します。  

`$ sudo yum install -y nginx`  

`$ sudo systemctl start nginx`  
ブラウザにて http://192.168.33.19 (Vagrantfileでipを書き換えた方はそのipアドレス)と入力し、NginxのWelcomeページが表示されましたでしょうか？  

`sudo vi /etc/nginx/conf.d/default.conf`  
編集範囲がやや広いですが頑張りましょう。  
コメント外しのミスが多いのでしっかりと確認してください。  
```
server {  
  listen       80;  
  server_name  192.168.33.10; # Vagranfileでコメントを外した箇所のipアドレスを記述してください。  
  # ApacheのDocumentRootにあたります  
  root /vagrant/laravel_app/public; # 追記  
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

### php-fpmの設定ファイルを編集
次に php-fpm の設定ファイルを編集していきます。  

`$ sudo vi /etc/php-fpm.d/www.conf`  
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

画面は表示されますが、以下のようなLaravelのエラーが表示されると思います。  

`The stream or file "/vagrant/laravel_app/storage/logs/laravel.log" could not be opened: failed to open stream: Permission denied`  

これは 先程php-fpmの設定ファイルの user と group を nginx に変更したと思いますが、ファイルとディレクトリの実行 user と group に nginx が許可されていないため起きているエラーです。  

以下のコマンドを実行して nginx というユーザーでもログファイルへの書き込みができる権限を付与してあげましょう。  
```
$ cd /vagrant/laravel_app  
$ sudo chmod -R 777 storage  
```
## laravel 6.0 認証機能

LaravelをInstallをしてProjectを作成  
`composer create-project laravel/laravel --prefer-dist laravel_app 6.0`  

以下のコマンドを実行しProjectのディレクトリに移動しサーバーを立ち上げます。  
```
cd laravel_app  
php artisan serve  
```
マイグレーションを実行しておきます。  
`$ php artisan migrate`

laravel/uiライブラリをインストールします。
`$ composer require laravel/ui 1.0`

認証機能に必要なファイルを追加します。  
`$ php artisan ui vue --auth`  

見た目を整えます。  
```
$ npm install
$ npm run dev
```
これで、環境構築できます。

環境構築の所感
***

仮想環境が、どういうものなのか理解するのに少し時間がかかってしまいました。  
OS上で別のOSを動作させることができるようになり、ネットワーク上に公開されているWebアプリケーションと同様のアプリケーションを自分のPC上で動かして
利用できるものだと解釈しました。（サーバーの代わり）  
今回は、nginxを使って環境構築をしましたが、エラーだらけで大変でした。  
特に辛かったエラーが  
`vagrant up`  
で起きたディレクトリが同期出来ないエラーと  
`The stream or file "/vagrant/laravel_app/storage/logs/laravel.log" could not be opened: failed to open stream: Permission denied`  
のエラーでした。  
Linuxについては少し触れたことがあったので、すんなりいけました。  
まだまだ基礎的なことしか触れられていないと思うので、理解をより深めていきたいです。  

参考サイト
***
[vagrant mount フォルダ共有ができない問題を解決する](https://omohikane.com/vagrant_mount_error_foldersync/)  
[Laravel キャッシュクリア系コマンドなど](https://qiita.com/Ping/items/10ada8d069e13d729701)  
[Laravel6 ログイン機能を実装する](https://qiita.com/ucan-lab/items/bd0d6f6449602072cb87)  
[認証 6.x Laravel - ReaDouble](https://readouble.com/laravel/6.x/ja/authentication.html)  
[Markdown記法 チートシート · GitHub](https://gist.github.com/mignonstyle/083c9e1651d7734f84c99b8cf49d57fa)  