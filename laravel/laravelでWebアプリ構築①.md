# LaravelでWebアプリケーションを作ってみる
## 【目的】
  - 社内的にレガシーフレームワークを刷新したいという声が上がっており、
    スクリプト言語をそもそも継続するべきなのかという問題はあるものの
    PHP界隈で最も使用されているフレームワークを技術検証含めて採用しようと考えた
## 【ゴール】
  - 新規プロジェクトの立ち上げと、ブラウザからトップ画面へのアクセス
## 【具体的な手順】
### 1. Composerでプロジェクトを新規作成する
LaravelのプロジェクトはComposerコマンドで作成する
```
composer create-project laravel/laravel sample_app
```
■ プロジェクトのファイル構成
```
  root/  
    |- app/             ・・・アプリケーションのロジック（MVCのControllerに当たる）  
    |- bootstrap/       ・・・laravelフレームワークの起動コード  
    |- config/          ・・・設定ファイル  
    |- database/        ・・・Migration（※後述）ファイルなどのDB関連ファイル  
    |- lang/            ・・・laravel9系ではroot直下、10系や8系ではresources直下に配置される  
    |- public/          ・・・Webサーバのドキュメントルート  
    |- resources/       ・・・ビューや言語変換用ファイル  
    |- routes/          ・・・ルーティング用ファイル   
    |- storage/         ・・・フレームワークが使用するファイル  
    |- tests/           ・・・テストコード  
    |- vendor/          ・・・Composerでインストールしたライブラリ  
    |- .env             ・・・環境変数ファイル　　　　　　　　　　　　　など  
```

### 2. データベースの設定を行う
  - プロジェクトのルートディレクトリに存在する「.env」ファイルを開く
  - デフォルトでは「mysql」が設定されているため、要件に合わせて変更する
    ```
    DB_CONNECTION=mysql
    DB_HOST=127.0.0.1
    DB_PORT=3306
    DB_DATABASE=laravel
    DB_USERNAME=root
    DB_PASSWORD=
    ```
  - ちなみに「config」フォルダ内にある「database.php」ファイルには、
    「.env」ファイルで定義した変数をデフォルトとして採用し、
    「.env」ファイルで定義されていない場合のデフォルト値を定義している

### 3. マイグレーションファイルを作成する
■ マイグレーションファイルの作成
  - マイグレーションファイルとは、データベースのバージョンコントロールのような機能であり、
    データベースのスキーマをチームで容易に共有できるようにする
    スキーマビルダと連携してデータベースのスキーマ作成を容易にする
  - 下記のコマンドでマイグレーションファイルを作成する
    ```
    php artisan make:migration create_name(s)_table --create=name(s)
    ```
    ※create_の後に付記するテーブル名は必ず複数形にすること
  - マイグレーションファイルは「database/migrations」ディレクトリに作成される
    デフォルトでは「up」と「down」メソッドが作成され、「upメソッド」にテーブル作成時の情報、
    「downメソッド」にマイグレーションを取り消す情報を記述する  
    ※書き方の見本は下記のサイトを参照
      ```
      https://readouble.com/laravel/5.0/ja/schema.html
      ```
    
■ マイグレーションの実行
  - 以下のコマンドを実行し、マイグレーションを行う
    ```
    php artisan migrate
    ```
  - テーブルが初期化されずに残っている場合は、キャッシュを削除してテーブルを一旦削除する
    ```
    php artisan cache:clear
    php artisan config:clear
    php artisan migrate:fresh
    ```

  
