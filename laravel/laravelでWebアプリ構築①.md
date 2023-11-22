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

### 4. モデルを作成する
  - モデルはテーブルとマッピングされたオブジェクトであり、DB操作を行うためのクラス
  - 下記のコマンドを実行してモデルファイルを作成する
    ```
    php artisan make:model Table_name
    ```
    ※Modelの名称は頭文字を大文字として、テーブル名と対応する単数形（Sを付けない）名称とする
  - 作成したモデルは「apps/Models」ディレクトリ直下に作成される
  - モデルは命名規則によってテーブルとマップされるので、テーブル名の単数形を名前に付けることで自動的にテーブルとマッピングされる
  - ちなみに、テーブル名とモデル名を対応させなくても、以下のように$tableプロパティで対応するテーブルを指定することが可能である
    ```
    protected $table = 'fruits';
    ```
  - 上記はモデルファイル内で必ず定義しないと後述するシーダーの実行や、
    DBの操作をする際にエラーになる（テーブル名が見つからない）

### 5. シーダーファイルを作成する（※任意）
■ シーダーファイル作成
  - シーダファイルは以下のコマンドで作成可能
    ```
    php artisan make:seeder NamesTableSeeder
    ```
  - 作成したシーダーファイルは「database/seeds」ディレクトリ直下に作成される
  - シーダーファイルの「runメソッド」内に登録する処理を記載する
  - 以下はサンプルのメソッド
    ```
    public function run()
    {
        // テーブルのクリア
        DB::table('persons')->truncate();
    
        // 初期データ用意（列名をキーとする連想配列）
        $persons = [
            ['name' => 'sample_user_01',
             'email' => 'test1@mail.co.jp',
             'birth_dt' => '1950/04/01'],
            ['name' => 'sample_user_02',
             'email' => 'test2@mail.co.jp',
             'birth_dt' => '1951/04/01'],
            ['name' => 'sample_user_03',
             'email' => 'test3@mail.co.jp',
             'birth_dt' => '1952/04/01']
        ];
    
        // 登録
        foreach($books as $book) {
          \App\Book::create($book);
        }
    }
    ```
  - また、DatabaseSeeder.phpに作成したSeederクラスが読み込まれるようにrun()メソッドに追加する
    ```
    public function run()
    {
        // PersonsTableSeederを読み込むように指定する
        $this->call(PersonsTableSeeder::class);
    }
    ```
  - DBを操作する方法は、①DBファサード、②クエリビルダ、③EloquentORMの３通りある
  - 以下はDBファサードを使用する場合に、クラスの冒頭にuseでファサードを呼び出す必要がある
    ```
    use Illuminate\Support\Facades\DB;
    ```
  - EloquentORMを使用する場合は、初めにuseでModelを呼び出しておく必要がある
    ```
    use App\Models\Person
    ・
    ・
    ・
    /**
     * トップ画面表示
     */
    public function index()
    {
        // Personsテーブルの一覧を取得
        $persons = Person::get();
        // Personのビューを表示する
        return view("person", compact("persons"));
    }
    
    ```
■ シーダーファイル実行
  - シーダーファイルは以下のコマンドで実行する
    ```
    php artisan db:seed
    ```

### 6. ルーティングの設定を行う
  - ルーティングの情報は「routes/web.php」に記載する
  - デフォルトではルートディレクトアクセス時のルーティング処理が記載されている
    ```
    Route::get('/', function () {
        return view('welcome');
    });
    ```
  - 以下のようにartisanコマンドでサーバーを立ち上げ、
    「http://localhost:8000/」にアクセスしてみるとwelcome.blade.phpの内容が表示される
    ```
    php artisan serve --port=8000
    ```
  - ルートパラメータについて、例えば以下のように記載した場合、
    「localhost:8000/book/1」にGETリクエストがきたら、BookControllerのshowメソッドに処理を振り分けて」という意味になる
    ```
    ■ Laravel8まで
    Route::get('book/{id}', 'BookController@show');
    ■ Laravel9以降
    use App\Http\Controllers\BookController;
    Route::get('book/{id}', [BookController::class, "index"]);
    ```
    BookControllerではshowメソッドに引数を持たせるようにする
    ```
    public function show($id)
    {
        return view('book', ['book' => Book::findOrFail($id)]);
    }
    ```
    ※上記の場合、ルーティングのパスに{id}を付与しているため、ポストパラメータとしてコントローラー側で受け取ることができる
    　フォームでPOSTするデータを受け取る場合は、functionの引数にpublic function show(Request $request){}のようにIllumintate\Http\Requestクラス
      を用いてリクエスト変数を取得して、$request->input("id")でPOSTパラメータを取得するか、
      Request::input("id", default_value);などのように直接Requestクラスのinput()メソッドを呼び出して取得する

### 7. コントローラーの作成
  - コントローラーはルーティングされてきたリクエストを受け取り、レスポンスを作成する
  - コントローラーを作成する際は以下のコマンドを実行する
    ```
    php artisan make:controller ControllerName
    ```
  - ちなみに下記のコマンドでコントローラーを作成すると、簡単にCRUDなどのメソッド(※)を立ち上げてくれる
    ```
    php artisan make:controller ExampleController --resource
    ```
    ※index、create、store、show、edit、update、destroy
  - コントローラーは「app/Http/Controllers」ディレクトリに作成される
  - ビューを表示するファンクション「public function index(){}」を定義する
  - コントローラのメソッドでビューを返したい場合はview()関数を使用する
  - view()関数は第1引数に表示したいビューの名称、第2引数にビューに渡したい値を設定する
  - compact関数は簡単に連想配列を渡すことができる。
    ```
    compact('book')　は　['book' => $book]　と同じ
    ```

### 8. ビューの作成
  - ビューは「resources/views」ディレクトリ直下に作成する
  - 名称は「view_name.blade.php」とする　（※１）
  - Larabelではビューの作成に「blade」というテンプレートエンジンを用いている
  - bladeでは{{ $variable }}と記述すると、コントローラから受け取った値を画面に出力できる
  - また、@foreach($array as $element)や@if($bool)とすることで制御構文を使用することもできる
  - 制御構文の終端は@endforeach、@endifと記述する
    ※bladeのマニュアルは以下を参照
    ```
    https://readouble.com/laravel/5.7/ja/blade.html
    ```
  - 作成後にartisanコマンドでサーバーを立ち上げて画面を確認する

### 9. 削除、更新機能の作成
  - 手順8で作成したviewに削除と更新機能を付ける
  - フォームタグでボタン設置行の利用者を削除する機能（delete）を以下の例を参考に作成
    ```
    <form action="/person/delete" method="post">
      @method("DELETE")
      @csrf
      <input type="hidden" name="id" value="{{ $person->id }}">
      <button type="submit" class="btn btn-xs btn-danger" aria-label="Left Align"><span class="glyphicon glyphicon-trash"></span></button>
    </form>
    ```
    - @method("DELETE")：HTTPのリクエスト方式（get、post、putなど）
    - @csrf：Bladeの標準ディレクティブであり、CSRF対策として、トークンのタグをhidden属性で埋め込むことができる
  - DBの削除、更新等のロジックを実装する場合、リターンでview()メソッドを返してしまうとURLが変わってしまうので、
    web.phpでindex()メソッドに定義しているルーティング名を呼び出すようにリダイレクトさせる
    ```
    [ web.php ]
    Route::get('person', [ExampleController::class, "index"])->name('examples.index');
    
    [ ExampleController.php ]
    return redirect()->route('examples.index');
    ```

## まとめ
・今回は、プロジェクト作成の基本からCRUDの一部実装までをざっくばらんにまとめた。
・DBを操作する方法がファサードやクエリビルダ、Eloquantなど覚えれば生SQLよりも圧倒的にすっきりとしたコーディングができると感じた。
・ルーティングやマイグレーション周りは今一つ理解が足りていないので、実際のプログラムを作成しながら習得できればと思う。

## 今後の課題
1. DB認証機能を実装してみる
2. ビューの継承を実装してみる
3. 画面の動的な生成をしてみる（React.js／Next.jsの導入）
4. 複雑なクエリによる一覧取得をしてみる
5. APIの呼び出しをしてみる

