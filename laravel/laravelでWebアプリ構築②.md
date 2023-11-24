# Larabelで認証機能を実装してみる
## 【概要】
  Laravelで認証機能を実装する場合、データベースにパスワードを保持して
  DB認証を行うか、Laravelの公式ライブラリで認証周りのひな型を作成できるパッケージを使用するか、
  いずれかの方法がまず考えられる。
  
## 1. Laravel公式ライブラリ（Laravel/ui）を使用する場合
■ 参考サイト
```
https://reffect.co.jp/laravel/laravel9-laravel-ui/
```
### 1-1. ライブラリをComposerでインストールする
  ```
  composer require laravel/ui
  ```
  ※2023/11/24時点で、laravel9のフレームワークの場合、v4.2.2がインストールされる
  
### 1-2. 認証画面の骨組み（Scaffold）を取得する
  ```
  php artisan ui bootstrap --auth
  ```
　※どのようなJavaScriptのライブラリがインストールされるのかpackage.jsonで確認することができます
 
### 1-3. 指示通りコマンドを実行する
　手順1-2を実行すると、「Please run [npm install && npm run dev] to compile your fresh scaffolding.」
  というメッセージが表示されるので、指示通りコマンドを実行する
  ```
  npm install && npm run dev 
  ```
  WindowsのVSCodeから実行する場合、上記のコマンドを実行しようとすると「&&」が無効となり
  実行できない場合がある。
  その場合は、コマンドを分けて実行すること

  ■ 自動で生成されるページ一覧
  1. ログインページ（resources/views/auth/login.blade.php）
  2. 新規ユーザー登録ページ（resources/views/auth/register.blade.php）
  3. メール認証ページ（	メール認証ページ）
  4. パスワード認証（resources/views/auth/passwords/confirm.blade.php）
  5. パスワード再設定（メアド入力画面）（resources/views/auth/passwords/email.blade.php）
  6. パスワード再設定（再設定画面）（resources/views/auth/passwords/reset.blade.php）
  7. ホーム画面（resources/views/home.blade.php）

  ■ ルーティング設定
  ```
  // ログイン関連のページのルーティング
  // これ1行で全ページ（ホーム画面以外）のルーティングが完了する
  Auth::routes();
  
  // ログインのホーム画面のルーティング
  Route::get('/home', [App\Http\Controllers\HomeController::class, 'index'])->name('home');
  ```

  ■ コントローラー
  1. ログインページのコントローラー（app/Http/Controllers/Auth/LoginController.php）
  2. 新規ユーザー登録ページのコントローラー（app/Http/Controllers/Auth/RegisterController.php）
  3. メール認証ページのコントローラー（app/Http/Controllers/Auth/VerificationController.php）
  4. パスワード認証ページのコントローラー（app/Http/Controllers/Auth/ConfirmPasswordController.php）
  5. パスワードを忘れた場合に、パスワード再設定のためのメール送信を行うコントローラー（app/Http/Controllers/Auth/ForgotPasswordController.php）
  6. パスワード再設定用のページを表示し、パスワードの変更を行うコントローラー（app/Http/Controllers/Auth/ResetPasswordController.php）
  7. ホーム画面のコントローラー（app/Http/Controllers/HomeController.php）

### 1-4. データベースをマイグレーションする
  下記コマンドを実行すると、認証に必要なテーブルが自動的に追加される
  ①users
  ②password_resets
  ```
  php artisan migrate
  ```
### 1-5. サーバを起動する
  ```
  php artisan serve --port=8000
  ```
  上記コマンドを実行すると、http://127.0.0.1:8000でサーバが起動するので、
  ブラウザからアクセスするとLaravelのデフォルトビュー画面が表示され、
  右上に「login」と「register」の２つのメニューリンクが表示される。

  ■ ポイント
  - php artisan serveコマンドでサーバを起動すると同時に
    npm run devコマンドでを起動しないと「http://localhost」にアクセスして、
    ログインやユーザー登録機能を確認しようとしても、下記のエラーが表示される
    「Start the development server. Run npm run dev in your terminal and refresh the page.」
　　→　別ターミナルで```npm run dev```と```php artisan serve```を実行する

### 1-6. 画面を確認してみる
  ```php artisan serve```コマンドで起動した「http://localhost:port」にアクセスし、
  「login」もしくは「register」メニューをクリックして、各画面が表示されることを確認する

### 1-7. 利用者登録してみる
  「register」メニューをクリックして、必要事項を入力したのち、
  登録ボタンをクリックするとログイン完了画面に遷移することを確認する

  登録された情報は、```php artisan migrate```コマンドを実行するとデフォルトで作成される
  「users」テーブルにレコードが登録される。

  ※登録先テーブルを変更したい場合
  - 参考サイト
    https://zenn.dev/miyapei/articles/725316f6c52a579f2f00
  - /App/Models/Users.php　の$table変数に対して、「protected $table = "登録先テーブル名";」
    を宣言することで登録先のテーブルを変更することができる。
  - オリジナルで作成したモデルを認証時に参照するようにしたい場合は、
    [/project_root/config/auth.php]ファイルの[provider]キーの[users]キーの「model」の値を変更する
  - 認証用テーブルの主キーが「id」以外の場合は、「protected $primaryKey = 'primary_key';」で指定する
  　また、主キーが数値の連番（increment変数）ではない場合は、「public $incrementing = false;」も指定すること
  - ログインアカウントに使用する情報（カラム）を変更する場合
    ベースは「\Illuminate\Foundation\Auth\AuthenticatesUsers」というトレイトの「username」という
    メソッドで管理されているため、「\App\Http\Controllers\Auth\LoginController」の中で、
    usernameメソッドをオーバーライド（再定義）することで実装が可能となる
    ```
    /**
     * Illuminate\Foundation\Auth\AuthenticatesUsers
     * 
     * Get the login username to be used by the controller.
     *
     * @return string
    */
    public function username() // このメソッドを追記
    {
        return 'login_account'; // 対象のカラム名にする
    }
    ```
  - ログインパスワードに使用する情報（カラム）を変更する場合
    パスワードカラムの指定は「Illuminate\Auth\Authenticatable」というトレイトがベースになっており、
    デフォルトのカラムは「password」となっている。
    変更する場合は、認証に利用するモデル（デフォルトだとUsers.php）に
    以下の記述を追加する
    ```
    /**
     * Get the password for the user.
     *
     * @return string
     */
    public function getAuthPassword() // これを追記
    {
        return $this->hash; // 対象のカラム名にする
    }
    ```

### 1-8. メール認証を必須にする
  - 変更１：「app/Models/User.php」
    ```
    use Illuminate\Contracts\Auth\MustVerifyEmail;
    
    class User extends Authenticatable implements MustVerifyEmail
    {
      ・
      ・
      ・
    }
    
    ```
    冒頭にuseでMustVerifyEmailを追加し、
    class名にimplements MustVerifyEmailを付記することで、
    MustVerifyEmailのインターフェースを利用することができる
    
  - 変更２：ルーティング「/project_root/routes/web.php」
    ```
    Route::get('/home', [App\Http\Controllers\HomeController::class, "index"])->middleware('verified')->name("home");
    ```
    上記の場合は、1ページ（各URL（コントローラクラス）のメソッド）ごとにアクセス制限をかけないといけない

    コントローラのコンストラクタで、一括でアクセス制限をかけることが可能である
    ```
    class HomeController extends Controller
    {
    		public function __construct()
        {
            // アクセス制限
            // ただしindex, show, topは除く
            $this->middleware('verified')->except(['index', 'show', 'top']);
        }
    }
    ```
    
## 2. Laravel公式スターターキット（Laravel/Breeze）を使用する場合
■ 参考サイト
```
https://readouble.com/laravel/9.x/ja/starter-kits.html
https://readouble.com/laravel/9.x/ja/authentication.html
```
### 2-1. composerでlaravel/Breezeをインストールする
  ```
  composer require laravel/breeze --dev
  ```
### 2-2. 認証周りのリソースをプロジェクトにインストールする
  認証ビュー、ルート、コントローラ、およびその他のリソースをアプリケーションにインストールする
  ```
  php artisan breeze:install (--dark ※ダークモードを適用させる場合)

  php artisan migrate
  npm install
  npm run dev
  ```
### 2-3. ログイン画面にアクセスしてみる
  http:://localhost:port/login や http://localhost:port/register
  にアクセスして、ログイン画面と登録画面が表示されることを確認する
  
## まとめ
  
## 課題
1. Laravelのサービスコンテナの概念を理解する
2. 依存性注入（DI）について理解する
3. php artisan serveではなく、別のミドルウェアでWebサーバを立ててアクセスできるようにする
