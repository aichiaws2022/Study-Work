#  Raisetech 第五回講義

##  課題の流れ

##  組み込みサーバーでの起動
　1. yumアップデート

```

$ sudo yum update -y

```
2.Node.jsやYarnなどの必要なパッケージをダウンロード
```

$ curl -fsSL https://rpm.nodesource.com/setup_16.x | sudo bash -
$ curl -sL https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
$ sudo yum -y install git gcc clang openssl-devel zlib-devel mysql-devel nodejs yarn

```
3. rbenv と ruby-build をインストール

```

$ curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash

```
4. rbenv のパスを通し、初期設定

```

$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
$ ~/.rbenv/bin/rbenv init
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
$ source .bash_profile

```
5. rubyをインストール

```

$ rbenv install -v 3.1.2
$ rbenv rehash
$ rbenv global 3.1.2
$ ruby -v

```
6. bundlerとrailsのインストール

```

$ gem install bundler:2.3.14
$ gem install rails 7.0.4

```
7. Mysqlのインストール

```

$ wget https://dev.mysql.com/get/mysql80-community-release-el7-7.noarch.rpm

$ sudo yum localinstall -y mysql80-community-release-el7-7.noarch.rpm

$ sudo yum install -y mysql-community-devel

$ sudo yum install -y mysql-community-server

```
8. サンプルアプリケーションのインストール

```

$ git clone https://github.com/yuta-ushijima/raisetech-live8-sample-app.git

```
9. config/databese.sample.ymlの編集

```

default: &default

adapter: mysql2

encoding: utf8mb4

pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

username: admin #RDSのユーザー名

password: #RDSのパスワード

host: #RDSのエンドポイント

```
10.必要なgemをインストール

```
$ bundle install
```
11. データベースの作成とマイグレーション

```
$ rails db:create
$ rails db:migrate
```
12. プリコンパイル

```
$ bundle exec rails assets:precompile RAILS_ENV=development
```
13. Railsサーバーの起動

```
$ rails s -b 0.0.0.0
```
##  nginxとunicornで起動
1. Nginxのインストール

```
$ sudo amazon-linux-extras install -y nginx1

$ sudo systemctl start nginx.service
```
2. unicornの設定確認

```bash

$ vi config/unicorn.rb

```

```

listen '/home/ec2-user/raisetech-live8-sample-app/unicorn.sock'

pid '/home/ec2-user/raisetech-live8-sample-app/unicorn.pid'　

```

4. Nginxの設定

```

$ sudo vi /etc/nginx/nginx.conf

```

以下設定

```

upstream unicorn {

# Unicornと連携させるための設定。

# config/unicorn.rb内のunicorn.sockを指定する

server unix:/home/ec2-user/raisetech-live8-sample-app/unicorn.sock;

}

server {

listen 80;

server_name インスタンスip;

client_max_body_size 100m;

# 接続が来た際のrootディレクトリ

root /home/ec2-user/raisetech-live8-sample-app/public;

try_files $uri/index.html $uri @unicorn;

location @unicorn {

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

proxy_set_header Host $http_host;

proxy_redirect off;

proxy_pass http://unicorn;

}

error_page 500 502 503 504 /500.html;

}

```
6. unicornの起動

```bash

$ bundle exec unicorn_rails -c config/unicorn.rb -D

```

----

##  ELBを通して動作



1. ターゲットグループとロードバランサーを作成

2. config/environments/development.rbに作成したALBのDNS追加

```

config.hosts << "ALBのDNS"

```
##  S3で画像保存

1. S3バケットの作成

2. IAMユーザーの作成

・ アクセス許可の設定で AmazonS3FullAccesポリシーをアタッチ、IAMポリシーで制御するため、バケットポリシーは今回使わない。

3. アプリケーション側で秘匿情報の設定

* config/credentials/development.yml.encを消去する

* 新たにcredentials.yml.encとkeyを作成し、アクセスキーの内容を記述する

```

$ EDITOR="vim" bin/rails credentials:edit -e development

```

```development.yml.enc

aws:

access_key_id: ******

secret_access_key: ******

```
4. config/storage.ymlの内容変更

~~~

amazon:

service: S3

access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>

secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>

region: バケットのリージョン

bucket: 作成したバケット名

~~~

