# 第1章: Laravelアプリケーションの作成

## 1.4 DockerのビルドとLaravelのインストール

+ `Makefile`を編集(P04〜)<br>

```:Makefile
up:
	docker-compose up -d
build:
	docker-compose build --no-cache --force-rm
laravel-install:
	docker-compose exec app composer create-project --prefer-dist "laravel/laravel=8.*" . // 編集
create-project:
	mkdir backend
	@make build
	@make up
	@make laravel-install
	docker-compose exec app php artisan key:generate
	docker-compose exec app php artisan storage:link
	docker-compose exec app chmod -R 777 storage bootstrap/cache
	@make fresh
install-recommend-packages:
	docker-compose exec app composer require doctrine/dbal "^2"
	docker-compose exec app composer require --dev ucan-lab/laravel-dacapo
	docker-compose exec app composer require --dev barryvdh/laravel-ide-helper
	docker-compose exec app composer require --dev beyondcode/laravel-dump-server
	docker-compose exec app composer require --dev barryvdh/laravel-debugbar
	docker-compose exec app composer require --dev roave/security-advisories:dev-master
	docker-compose exec app php artisan vendor:publish --provider="BeyondCode\DumpServer\DumpServerServiceProvider"
	docker-compose exec app php artisan vendor:publish --provider="Barryvdh\Debugbar\ServiceProvider"
init:
	docker-compose up -d --build
	docker-compose exec app composer install
	docker-compose exec app cp .env.example .env
	docker-compose exec app php artisan key:generate
	docker-compose exec app php artisan storage:link
	docker-compose exec app chmod -R 777 storage bootstrap/cache
	@make fresh
remake:
	@make destroy
	@make init
stop:
	docker-compose stop
down:
	docker-compose down --remove-orphans
restart:
	@make down
	@make up
destroy:
	docker-compose down --rmi all --volumes --remove-orphans
destroy-volumes:
	docker-compose down --volumes --remove-orphans
ps:
	docker-compose ps
logs:
	docker-compose logs
logs-watch:
	docker-compose logs --follow
log-web:
	docker-compose logs web
log-web-watch:
	docker-compose logs --follow web
log-app:
	docker-compose logs app
log-app-watch:
	docker-compose logs --follow app
log-db:
	docker-compose logs db
log-db-watch:
	docker-compose logs --follow db
web:
	docker-compose exec web ash
app:
	docker-compose exec app bash
migrate:
	docker-compose exec app php artisan migrate
fresh:
	docker-compose exec app php artisan migrate:fresh --seed
seed:
	docker-compose exec app php artisan db:seed
dacapo:
	docker-compose exec app php artisan dacapo
rollback-test:
	docker-compose exec app php artisan migrate:fresh
	docker-compose exec app php artisan migrate:refresh
tinker:
	docker-compose exec app php artisan tinker
test:
	docker-compose exec app php artisan test
optimize:
	docker-compose exec app php artisan optimize
optimize-clear:
	docker-compose exec app php artisan optimize:clear
cache:
	docker-compose exec app composer dump-autoload -o
	@make optimize
	docker-compose exec app php artisan event:cache
	docker-compose exec app php artisan view:cache
cache-clear:
	docker-compose exec app composer clear-cache
	@make optimize-clear
	docker-compose exec app php artisan event:clear
npm:
	@make npm-install
npm-install:
	docker-compose exec web npm install
npm-dev:
	docker-compose exec web npm run dev
npm-watch:
	docker-compose exec web npm run watch
npm-watch-poll:
	docker-compose exec web npm run watch-poll
npm-hot:
	docker-compose exec web npm run hot
yarn:
	docker-compose exec web yarn
yarn-install:
	@make yarn
yarn-dev:
	docker-compose exec web yarn dev
yarn-watch:
	docker-compose exec web yarn watch
yarn-watch-poll:
	docker-compose exec web yarn watch-poll
yarn-hot:
	docker-compose exec web yarn hot
db:
	docker-compose exec db bash
sql:
	docker-compose exec db bash -c 'mysql -u $$MYSQL_USER -p$$MYSQL_PASSWORD $$MYSQL_DATABASE'
redis:
	docker-compose exec redis redis-cli
ide-helper:
	docker-compose exec app php artisan clear-compiled
	docker-compose exec app php artisan ide-helper:generate
	docker-compose exec app php artisan ide-helper:meta
	docker-compose exec app php artisan ide-helper:models --nowrite
```

+ `$ make create-project`を実行<br>

+ localhostにアクセスしてLaravelの初期画面が表示されればOK<br>

### 1.5.1 Laravel Breezeのインストール

+ `$ docker compose exec app composer require laravel/breeze:1.10 --dev`を実行<br>

### 1.5.2 php artisan breeze:install の実行

+ `$ docker compose exec app php artisan breeze:install`を実行<br>

### LaravelからViteをアンインストール

+ `docker compose exec web npm remove vite`を実行<br>

+ `docker compose exec web npm remove laravel-vite-plugin`を実行<br>

+ `backend/vite.config.js`を削除<br>

+ `backend/resources/views/layouts/app.blade.php`を編集<br>

```html:app.blade.php
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">

<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <title>{{ config('app.name', 'Laravel') }}</title>

    <!-- Fonts -->
    <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700&display=swap">

    <!-- Styles -->
    <link rel="stylesheet" href="{{ asset('css/app.css') }}">

    <!-- Scripts -->
    <script src="{{ asset('js/app.js') }}" defer></script>
</head>

<body class="font-sans antialiased">
    <div class="min-h-screen bg-gray-100">
        @include('layouts.navigation')

        <!-- Page Heading -->
        <header class="bg-white shadow">
            <div class="max-w-7xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
                {{ $header }}
            </div>
        </header>

        <!-- Page Content -->
        <main>
            {{ $slot }}
        </main>
    </div>
</body>

</html>
```

+ `backend/resources/views/layouts/guest.blade.php`を編集<br>

```html:guest.blade.php
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">

<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <title>{{ config('app.name', 'Laravel') }}</title>

    <!-- Fonts -->
    <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700&display=swap">

    <!-- Styles -->
    <link rel="stylesheet" href="{{ asset('css/app.css') }}">

    <!-- Scripts -->
    <script src="{{ asset('js/app.js') }}" defer></script>
</head>

<body>
    <div class="font-sans text-gray-900 antialiased">
        {{ $slot }}
    </div>
</body>

</html>
```

+ `backend/resources/app.js`を編集<br>

```js:app.js
require('./bootstrap')

require('alpinejs')
```

+ `$ docker compose exec web npm install laravel-mix --save-dev`(webbapck.mix.jsがない場合)を実行<br>

+ `backend/resources/css/app.css`を編集<br>

```css:app.css
@import 'tailwindcss/base';
@import 'tailwindcss/components';
@import 'tailwindcss/utilities';
```

+ `backend/webpack.mix.js`を編集<br>

```js:webpack.mix.js
const mix = require('laravel-mix')

/*
 |--------------------------------------------------------------------------
 | Mix Asset Management
 |--------------------------------------------------------------------------
 |
 | Mix provides a clean, fluent API for defining some Webpack build steps
 | for your Laravel applications. By default, we are compiling the CSS
 | file for the application as well as bundling up all the JS files.
 |
 */

mix
  .js('resources/js/app.js', 'public/js')
  .postCss('resources/css/app.css', 'public/css', [
    require('postcss-import'),
    require('tailwindcss'),
    require('autoprefixer'),
  ])

mix.webpackConfig({
  stats: {
    children: true,
  },
})
```

+ `backend/package.json`を編集<br>

```json:package.json
{
    "private": true,
    "scripts": {
        "dev": "npm run development",
        "development": "mix",
        "watch": "mix watch",
        "watch-poll": "mix watch -- --watch-options-poll=1000",
        "hot": "mix watch --hot",
        "prod": "npm run production",
        "production": "mix --production"
    },
    "devDependencies": {
        "@tailwindcss/forms": "^0.5.2",
        "alpinejs": "^3.4.2",
        "autoprefixer": "10.4.5", // 編集
        "axios": "^0.21",
        "laravel-mix": "^6.0.49",
        "lodash": "^4.17.19",
        "postcss": "^8.4.6",
        "tailwindcss": "^3.1.0"
    }
}
```

+ `backend/package.lock.json`を削除<br>

### 1.5.3 Node.js関連モジュールのインストール

+ `$ docker compose exec web npm install`を実行<br>

+ `$ docker compose exec web npm run dev`を実行<br>

### 1.5.4 .gitignoreの編集

+ `backend/.gitignore`を編集<br>

```:.gitignore
/node_modules
/public/css
/public/hot
/public/js
/public/mix-manifest.json
/public/storage
/storage/*.key
/vendor
.env
.env.backup
.phpunit.result.cache
docker-compose.override.yml
Homestead.json
Homestead.yaml
npm-debug.log
yarn-error.log
```

### 1.5.6 認証機能の動作確認

+ localhostにアクセスし、ユーザー登録を実施してみる<br>

## 1.6: 既存のGitHub Actionsのワークフローの削除

+ `.github/workflows/laravel-create-project.yml`を削除<br>

+ `.github/workflows/laravel-git-clone.yml`を削除<br>

# 第2章 Terraformのセットアップ(P09〜)

## 2.2 Terraformのインストール

+ `$ brew install terraform`を実行<br>

## 2.3 tfenvの利用

### 2.3.1 tfenvを使う理由

Terraform でインフラを一度新規構築した後、クライアントで使用する Terraform のバージョンを上げると、<br>
Terraform の記法や利用できる機能が変わったことを理由として、作成済みの Terraform のコード (tf ファイル) を一部書き換える必要が出てくる場合があります。<br>
そのため、一度構築したインフラは、いったんは新規構築時に使った Terraform のバージョンで保守することになります。<br>
もし、利用する Terraform のバージョンを上げる際は、移行計画を別途立ててコードの書き換えを行うことになるかと思います。<br>
つまり、クライアントでは常時最新バージョンの Terraform を使うのではなく、特定のバージョンに固定して使う必要があります。<br>
一方で、複数プロジェクトで Terraform を使用している場合、プロジェクトごとに Terraform のバージョンは異なるでしょうから、クライアントで使用する Terraform は複数のバージョンから切り替えることができないと不便です。<br>
そこで、tfenv をインストールします。<br>

### 2.3.2 tfenvのインストール

+ `$ brew install tfenv`を実行<br>

## 2.3.3 tfenvでインストール可能なTerraformバージョンnの確認

+ `$ tfenv list-remote`を実行<br>

```:terminal
1.4.0-alpha20221109
1.3.4
1.3.3
1.3.2
1.3.1
1.3.0
1.3.0-rc1
1.3.0-beta1
1.3.0-alpha20220817
1.3.0-alpha20220803
1.3.0-alpha20220706
1.3.0-alpha20220622
1.3.0-alpha20220608
1.2.9
1.2.8
1.2.7
1.2.6
1.2.5
1.2.4
1.2.3
1.2.2
1.2.1
1.2.0
1.2.0-rc2
1.2.0-rc1
1.2.0-beta1
1.2.0-alpha20220413
1.2.0-alpha-20220328
1.1.9
1.1.8
1.1.7
1.1.6
1.1.5
1.1.4
1.1.3
1.1.2
1.1.1
1.1.0
1.1.0-rc1
1.1.0-beta2
1.1.0-beta1
1.1.0-alpha20211029
1.1.0-alpha20211020
1.1.0-alpha20211006
1.1.0-alpha20210922
1.1.0-alpha20210908
1.1.0-alpha20210811
1.1.0-alpha20210728
1.1.0-alpha20210714
1.1.0-alpha20210630
1.1.0-alpha20210616
1.0.11
1.0.10
1.0.9
1.0.8
1.0.7
1.0.6
1.0.5
1.0.4
1.0.3
1.0.2
1.0.1
1.0.0
0.15.5
0.15.4
0.15.3
0.15.2
0.15.1
0.15.0
0.15.0-rc2
0.15.0-rc1
0.15.0-beta2
0.15.0-beta1
0.15.0-alpha20210210
0.15.0-alpha20210127
0.15.0-alpha20210107
...(省略)...
```

### 2.3.4 tfenvを使ったTerraformのインストール

+ `tfenv install 1.0.0`を実行<br>

## 2.3.5 インストール済み及び使用中のバージョンの確認

+ `$ tfenv list`を実行<br>

```:terminal
  1.0.0
No default set. Set with 'tfenv use <version>'
```

### 2.3.5 使用するバージョンの切り替え

+ `$ tfenv use 1.0.0`を実行<br>

```:terminal
Switching default version to v1.0.0
Default version (when not overridden by .terraform-version or TFENV_TERRAFORM_VERSION) is now: 1.0.0
```

+ `$ tfenv list`を実行<br>

```:terminal
* 1.0.0 (set by /usr/local/Cellar/tfenv/3.0.0/version)
```

## 2.4: AWS CLIのインストールとプロファイルの設定(P12〜)

### 2.4.1 AWS CLIのインストール

+ `$ aws --version`を実行<br>

コマンド実行後、以下のようにバージョンが表示されれば、AWS CLI はインストール済みです。<br>

```:terminal
aws-cli/2.1.10 Python/3.9.1 Darwin/19.6.0 source/x86_64 prompt/off
```

もし、AWS CLI がインストールされていなければ、以下コマンドを実行してください。<br>

+ `$ brew install awscli`を実行<br>

インストール完了後、以下のコマンドを実行してください。<br>

+ `$ aws --version`を実行<br>

```:terminal
aws-cli/2.8.12 Python/3.10.8 Darwin/19.6.0 source/x86_64 prompt/off
```
