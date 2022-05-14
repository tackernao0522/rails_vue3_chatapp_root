# 01 Rails を動かす Dockerfile を作成する 1/2 〜作成編〜

- `$ mkdir {api,front}(スペースは空けない)`を実行<br>

- api ディレクトリに移動する <br>

* `$ touch {Dockerfile,Gemfile,Gemfile.lock}`を実行<br>

- `api/Gemfile`を編集<br>

```:Gemfile
source 'https://rubygems.org'
gem 'rails', '~> 6.0.4' # 6.0.0以上、6.1.0未満で最新 = 6.0.x
```

- `api/Dockerfile`を編集<br>

```:Dockerfile
FROM node:16.13.1-alpine as node

FROM ruby:2.6.4-alpine

COPY --from=node /usr/local/bin/node /usr/local/bin/node

ARG WORKDIR

ARG RUNTIME_PACKAGES="tzdata postgresql-dev postgresql git"

ARG DEV_PACKAGES="build-base curl-dev"

ENV HOME=/${WORKDIR} \
    LANG=C.UTF-8 \
    TZ=Asia/Tokyo

WORKDIR ${HOME}

COPY Gemfile* ./

RUN apk update && \
  apk upgrade && \
  apk add --no-cache ${RUNTIME_PACKAGES} && \
  apk add --virtual build-dependencies --no-cache ${DEV_PACKAGES} && \
  bundle install -j4 && \
  apk del build-dependencies

COPY . ./

CMD ["rails", "server", "-b", "0.0.0.0"]
```

# 02 Vue.jsを動かすDockerfileを作成する

+ `$ cd front`を実行<br>

+ `$ touch Dockerfile`を実行<br>

```:Dockerfile
# FROM node:14.4.0-alpine 使用不可
FROM node:16.13.1-alpine

ARG WORKDIR
ARG API_URL

ENV HOME=/${WORKDIR} \
  LANG=C.UTF-8 \
  TZ=Asia/Tokyo \
  # これを指定しないとブラウザからhttp://localhost へアクセスすることができない。
  # コンテナのNuxt.jsをブラウザから参照するためにip:0.0.0.0に紐付ける
  # https://ja.nuxtjs.org/faq/host-port/
  HOST=0.0.0.0 \
  API_URL=${API_URL}

WORKDIR ${HOME}

# ローカル上のpackageをコンテナにコピーしてインストールする
# COPY package*.json ./
RUN apk update && npm install -g npm

COPY . ./

RUN npm run build

CMD ["npm", "run", "dev"]
```

# 03 env, .gitignore, docker-compose.yml を作成

+ `root $ touch {.gitignore,.env,docker-compose.yml}`を実行<br>

+ `root .gitignore`に追記<br>

```.gitignore
/.DS_store
/.DS_Store
```

+ `root .env`を編集<br>

```:.env
# commons
WORKDIR=app
API_PORT=8000
FRONT_PORT=3000

# db
POSTGRES_PASSWORD=password
```

+ `root docker-compose.yml`を編集<br>

```yml:docker-compose.yml
# composeファイルのバージョン指定
# Doc: https://docs.docker.com/compose/compose-file/compose-versioning/
version: "3.8"

services:
# サービス(= コンテナ)
  db:
    # ベースイメージを定義
    image: postgres:13.1-alpine
    # 環境変数を定義
    environment:
      # OSのタイムゾーン
      TZ: UTC
      # postgresのタイムゾーン
      PGTZ: UTC
      # データベースのパスワード
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
    # ホスト側のディレクトリをコンテナで使用する
    # volumes: ホストパス(絶対 or 相対) : コンテナパス(絶対)
    volumes:
      - "./api/tmp/db:/var/lib/postgresql/data"

  api:
    # ベースイメージとなるDockerfileを指定
    build:
      context: ./api
      # Dockerfileに変数を渡す
      args:
        WORKDIR: $WORKDIR
    environment:
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
      API_DOMAIN: "localhost:$FRONT_PORT"       # 追加
      BASE_URL: "http://localhost:$API_PORT"
    # command: /bin/sh -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 8000 -b '0.0.0.0'"
    volumes:
      - "./api:/$WORKDIR"
    # サービスの依存関係を定義(起動の順番)
    # 公開したいポート番号:コンテナポート
    depends_on:
      - db
    # 公開用ポートを指定
    ports:
      - "$API_PORT:3000"

  front:
    build:
      context: ./front
      args:
        WORKDIR: $WORKDIR
        API_URL: "http://localhost:$API_PORT" # 追加
    # コンテナで実行したいコマンド(CMD)
    # command: npm run serve
    volumes:
      - "./front:/$WORKDIR"
    ports:
      - "$FRONT_PORT:3000"
    depends_on:
      - api
```

# 04 Rails アプリを立ち上げる

+ `root docker compose build`を実行<br>

+ `$ root docker images`を実行してイメージが出来ているか確認<br>

```:terminal
REPOSITORY             TAG       IMAGE ID       CREATED              SIZE
chat-rails-vue_front   latest    6b7661da9e77   35 seconds ago       344MB
chat-rails-vue_api     latest    619f1daae594   About a minute ago   258MB
```

+ `$ docker compose run --rm api rails new . -f -B -d postgresql --skip-bundle --skip-turbolinks --skip-test --api`を実行<br>

+ `$ docker compose build`を実行<br>

+ `api/config/database.yml`を編集<br>

```yml:database.yml
# PostgreSQL. Versions 9.3 and up are supported.
#
# Install the pg driver:
#   gem install pg
# On macOS with Homebrew:
#   gem install pg -- --with-pg-config=/usr/local/bin/pg_config
# On macOS with MacPorts:
#   gem install pg -- --with-pg-config=/opt/local/lib/postgresql84/bin/pg_config
# On Windows:
#   gem install pg
#       Choose the win32 build.
#       Install PostgreSQL and put its /bin directory on your path.
#
# Configure Using Gemfile
# gem 'pg'
#
default: &default
  adapter: postgresql
  encoding: unicode
  host: db # 追加
  username: postgres # 追加
  password: <%= ENV["POSTGRES_PASSWORD"] %> # 追加
  # For details on connection pooling, see Rails configuration guide
  # https://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: app_development

  # The specified database role being used to connect to postgres.
  # To create additional roles in postgres see `$ createuser --help`.
  # When left blank, postgres will use the default role. This is
  # the same name as the operating system user that initialized the database.
  #username: app

  # The password associated with the postgres role (username).
  #password:

  # Connect on a TCP socket. Omitted by default since the client uses a
  # domain socket that doesn't need configuration. Windows does not have
  # domain sockets, so uncomment these lines.
  #host: localhost

  # The TCP port the server listens on. Defaults to 5432.
  # If your server runs on a different port number, change accordingly.
  #port: 5432

  # Schema search path. The server defaults to $user,public
  #schema_search_path: myapp,sharedapp,public

  # Minimum log levels, in increasing order:
  #   debug5, debug4, debug3, debug2, debug1,
  #   log, notice, warning, error, fatal, and panic
  # Defaults to warning.
  #min_messages: notice

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  <<: *default
  database: app_test

# As with config/credentials.yml, you never want to store sensitive information,
# like your database password, in your source code. If your source code is
# ever seen by anyone, they now have access to your database.
#
# Instead, provide the password as a unix environment variable when you boot
# the app. Read https://guides.rubyonrails.org/configuring.html#configuring-a-database
# for a full rundown on how to provide these environment variables in a
# production deployment.
#
# On Heroku and other platform providers, you may have a full connection URL
# available as an environment variable. For example:
#
#   DATABASE_URL="postgres://myuser:mypass@localhost/somedatabase"
#
# You can use this database configuration with:
#
#   production:
#     url: <%= ENV['DATABASE_URL'] %>
#
production:
  <<: *default
  database: app_production
  username: app
  password: <%= ENV['APP_DATABASE_PASSWORD'] %>
```

* `$ docker compose up api`を実行<br>

+ `root $ docker compose run --rm api rails db:create`を実行<br>

+ localhost:8000 にアクセスして初期画面が表示されればOK<br>

# 05 Vue.jsプロジェクトの新規作成

+ `$ root $ docker compose run --rm front sh`を実行<br>

+ `/app # npm init vite@latest`を実行<br>

+ `下記の質問に答える

```:terminal
Need to install the following packages:
  create-vite@latest
Ok to proceed? (y) y
✔ Project name: … vite-project
✔ Select a framework: › vue
✔ Select a variant: › vue

Scaffolding project in /app/vite-project...

Done. Now run:

  cd vite-project
  npm install
  npm run dev
```

+ `/app # la -a`を実行してコンテナ内のディレクトリ/ファイルを確認<br>

+ 生成されたファイル群を移動させて、要らないvite-projectディレクトリを削除<br>

+ `/app # mv vite-project/* /app/`を実行<br>

+ `/app # mv vite-project/.gitignore /app/`を実行<br>

+ `/app # mv vite-project/.vscode /app/`を実行<br>

+ `/app # rmdir vite-project`を実行<br>

+ `/app # npm install`を実行<br>

+ `/app # exit`を実行してfrontコンテナから抜ける<br>

+ `front/vite.config.js`を編集<br>

```js:vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue()],
  // 追記
  server: {
    host: true
  }
})
```

+ `localhost:3000`にアクセスして初期画面が表示されればOK<br>

+ `root $ docker compose run --rm front npm install vue-router@4`を実行してvue-routerをインストール<br>

+ vite-vue-routerについて https://zenn.dev/takpon1751/articles/83135cc6a0fc9f <br>

