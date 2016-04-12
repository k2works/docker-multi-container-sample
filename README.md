# 目的
DockerMultiContanierを使ったRailsアプリケーションサンプルをセットアップする

# 前提
| ソフトウェア     | バージョン    | 備考         |
|:---------------|:-------------|:------------|
| docker         | 1.10.3       |             |
| docker-compose | 1.6.2        |             |
| vagrant        | 1.7.4        |             |

+ DockerHubのアカウントを作っている

# 構成

+ 開発環境構築
+ アプリケーション実行

# 詳細
## 開発環境構築
### 実行環境起動
```
$ vagrant up
$ vagrant ssh
$ cd /vagrant/
$ mkdir sample-app
$ cd sample-app
```

### アプリケーションのセットアップ
#### Railsアプリケーションのセットアップをする
```
$ docker run -it --rm --user "$(id -u):$(id -g)" -v "$PWD":/usr/src/app -w /usr/src/app rails rails new --skip-bundle .
$ docker run --rm -v "$PWD":/usr/src/app -w /usr/src/app ruby:2.3 bundle install
```

`Dockerfile-rails`を作成して以下のコマンドを実行する
```
$ cp ../Dockerfile-rails .
$ docker build -t my-rails-app -f Dockerfile-rails .
$ docker run --name some-rails-app -p 3000:3000 -d my-rails-app
```
http://127.0.0.1:3000/ に接続して動作を確認する。確認が終わったらコンテナを停止削除する
```
$ docker stop some-rails-app
$ docker rm some-rails-app
```

以下の内容`docker-compose.yml`を作成する
```
version: '2'
services:
  web:
    image: my-rails-app
    volumes:
      - .:/usr/src/app
    ports:
      - "3000:3000"
```
docker-composeを実行する
```
$ docker-compose up
```
動作を確認したらCtr-cで終了する

#### MySQLサーバーのセットアップをする
`Dockerfile-mysql`を作成して以下のコマンドを実行する

```
$ cp ../Dockerfile-mysql .
$ docker build -t my-rails-app-mysql -f Dockerfile-mysql .
```
`config/database.yml`を編集する

```
$ cp ../container/web/config/database.yml ./config/
```
`Gemfile`を編集してコンテナを再ビルドする
```
# Use mysql as the database for Active Record
gem 'mysql2', '>= 0.3.13', '< 0.5'
```

```
$ docker run --rm -v "$PWD":/usr/src/app -w /usr/src/app ruby:2.3 bundle install
$ docker build -t my-rails-app -f Dockerfile-rails .
```

`docker-compose.yml`を編集する

```
version: '2'
services:
  db:
    image: my-rails-app-mysql
  web:
    image: my-rails-app
    volumes:
      - .:/usr/src/app
    ports:
      - "3000:3000"
    depends_on:
      - db
```
docker-composeを実行する
```
$ docker-compose up
```
動作を確認したらCtr-cで終了する

#### Nginxサーバーのセットアップをする
`proxy`ディレクトを追加する

```
$ cp -r ../container .
```

`Dockerfile-nginx`を作成して以下のコマンドを実行する

```
$ cp ../Dockerfile-nginx .
$ docker build -t my-rails-app-nginx -f Dockerfile-nginx .
```

`docker-compose.yml`を編集する

```
version: '2'
services:
  db:
    image: my-rails-app-mysql
  web:
    image: my-rails-app
    volumes:
      - .:/usr/src/app
    ports:
      - "3000:3000"
    depends_on:
      - db
  proxy:
    image: my-rails-app-nginx
    volumes:
      - ./container/proxy/nginx.conf:/etc/nginx/nginx.conf
      - ./container/proxy/conf.d/default.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "8080:80"
    depends_on:
      - web            
```
docker-composeを実行する
```
$ docker-compose up
```
http://127.0.0.1:8080/ に接続して動作を確認する
動作を確認したらCtr-cで終了する

#### コンテナイメージをDockerHubにプッシュする
DockerHubにログインする
```
$ docker login
```
webイメージを作成してプッシュする
```
$ docker build -t k2works/my-rails-app -f Dockerfile-rails .
$ docker push k2works/my-rails-app
```
dbイメージを作成してプッシュする
```
$ docker build -t k2works/my-rails-app-mysql -f Dockerfile-mysql .
$ docker push k2works/my-rails-app-mysql
```
proxyイメージを作成してプッシュする
```
$ docker build -t k2works/my-rails-app-nginx -f Dockerfile-nginx .
$ docker push k2works/my-rails-app-nginx
```

#### docker-composeをDockerHubからプルして実行できるようにする
docker-compose.ymlを編集する
```
$ cp ../docker-compose.yml .
```
コンテナイメージを初期化して再実行する
```
$ docker rm -v $(docker ps -a -q)
$ docker rmi $(docker images -q)
$ docker-compose up
```

#### 後片付け
```
$ exit
$ vagrant destroy
```

# 参照
+ [DockerHub rails](https://hub.docker.com/_/rails/)
+ [DockerHub mysql](https://hub.docker.com/_/mysql/)
+ [DockerHub](https://hub.docker.com/)
