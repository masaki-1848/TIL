# LaravelにReactを導入してみる
## 【概要】
　Laravelは世界１位のシェアを誇るPHPのフレームワークであり、
  Webアプリケーションの開発を効率化してくれるものであるが、
  Reactは、シングルページアプリケーションやモバイルアプリ開発のために作られたJavascriptの
  ライブラリの１つである。
## 【導入手順】
### 1. composerでlaravelのプロジェクトにreactをインストールする
  ```
  ## 認証関連機能の雛型を提供するLaravel公式のライブラリを取得する
  composer require laravel/ui
  ## Reactの認証画面の骨組みを取得する
  php artisan ui react --auth
  ## nodejsのパッケージをインストールする
  npm install
  ## 開発環境でnodejsを実行する
  npm run dev
  ```
### 2. シングルページ用のphpファイルを残し、他は除外する
  「resources/views」フォルダ配下にはapp.blade.phpファイルのみがある状態とし、ファイル内には下記を記載する
  ```
  <!doctype html>
  <html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
  <head>
      <meta charset="utf-8">
      <meta name="viewport" content="width=device-width, initial-scale=1">
      <!-- CSRF Token -->
      <meta name="csrf-token" content="{{ csrf_token() }}">
      <title>{{ config('app.name', 'Laravel') }}</title>
      <!-- Scripts -->  
      <script src="{{ asset('js/app.js') }}" defer></script>
      <!-- Fonts -->
      <link rel="dns-prefetch" href="//fonts.gstatic.com">
      <link href="https://fonts.googleapis.com/css?family=Nunito" rel="stylesheet">
      <!-- Styles -->
      <link href="{{ asset('css/app.css') }}" rel="stylesheet">
  </head>
  <body>
      <div id="app"></div>
  </body>
  </html>
  ```
### 3. resources/js/components/Example.jsを修正する
  app.blade.phpの「id=”app”」部分に、Exampleコンポーネントをレンダリングするよう修正する
  ```
  if (document.getElementById('app')) {
    ReactDOM.render(<Example />, document.getElementById('app'));
  }
  ```
### 4. routes/web.phpを修正する
  ```
  Route::get('{any}', function () {
    return view('app');
  })->where('any','.*');
  ```
### 5. 再度ビルドする
  Laravelプロジェクトのルートディレクトリで、以下のコマンドを実行する
  ```
  npm run dev
  ```
  サイトにアクセスし、Reactが表示されていれば問題なく導入できている

## まとめ
  今回は、LaravelのフレームワークにReactを導入する簡易な方法を検証した。
　実際にSPAを開発してみなければ、わからないが、少なくとも環境設定そのものは非常に用意であり、
  初心者でも挑戦するハードルは低いと思われる。

## 課題
1. Dockerで開発環境を作ってみる
   - チーム開発を行う際に、簡単に開発環境を共有できるようにするため
2. SPAのアプリケーションを作ってみる
   - 一覧表示画面、登録画面等における表示切替が素早くできるのか確認する
3. 既存プロジェクトの再現
   - 既存プロジェクトの一部をSPA化して、導入の有用性を検証する
   - モダンフレームワークの効果を実感する
