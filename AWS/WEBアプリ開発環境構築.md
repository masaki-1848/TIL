# AWSのWEBアプリケーション王道パターン
## １、EC2+RDS
  ### 作業手順
  1. Publicサブネット内にElastic Compute Cloud(EC2)インスタンスを構築し、その中にWEBサーバのミドルウェアをインストールし、Privateサブネット内にRDS（Relational Database Service）でSQLサーバを構築する。
  2. RDSに対するセキュリティグループのインバウンドルールとして、Postgresqlの場合は、「ポート：5432」、ソースは「EC2インスタンスのセキュリティグループ」として
　　管理者ユーザからEC2インスタンスを踏み台として、RDSのプライベートサブネットにアクセスできるように設定する。
　　複数名で管理を行う場合は、社内ネットワークのグローバルＩＰを通信許可するか、踏み台サーバ（プロキシサーバ）からのアクセスのみを許可する設定を行う
  3. EC2に対するセキュリティグループのインバウンドルールとして、テスト環境ではHTTP(Port:80)とSSH(Port:22)、号口環境ではHTTPS(Port:443)を許可する設定とする。
  4. RDSのインスタンスを起動する際に、初期のデータベース名を指定しなければEC2からRDSにアクセスするときのデータベース名が不明となる（おそらく確認する方法はあるが）
 
 ### 操作時のポイント
  - RDSを検証用に作成する際に、インスタンスを「簡単に設定」を選択してしまうと、事前に作成したセキュリティグループが選択できなくなり、defaultで作成されるセキュリティグループが適用される
  - EC2インスタンスをスポットにすると、一時停止ができないため、都度作り直しになってしまう（Ipv4のパブリックIPが都度変わる）
  - 機能の要件にもよるが、データをクラウド上に補完する場合は保管する左記の国の法律上、情報開示ができなくなったり、取得できなかった利する可能性があるため、リージョンは注意すること
  - SFTPによるファイルのやり取りを行う場合、SSHに利用した公開鍵をppkからpemに変換する必要がある
    ※Puttygenを使用する場合は、「Load」でppkファイルを選択し、Conversionsタブから「Export OpenSSH Key」を選択するとpemファイルが出力される

 ### 課題
  - ソースをGitLabやGitHubから持ってくるときに、プライベートリポジトリとなると認証が必要となるため、うまく取得することができない

## CodeCommitを用いたEC2のデプロイメント構築
 ### 【参考サイト】
       https://docs.aws.amazon.com/ja_jp/codepipeline/latest/userguide/tutorials-simple-codecommit.html
 ### 【手順】
  #### 1.CodeCommitにアクセスし、create Repositoryから新規リポジトリを作成する
  リージョンは作業用リージョンに切り替えること
  #### 2.Gitの認証情報を作成する
  IAMのユーザ管理サービスから、git cloneする際に認証ユーザーを検索し、管理画面から「セキュリティ認証情報」タブを選択して、
  下部にある「AWS CodeCommit の HTTPS Git 認証情報 」セクションより、認証情報を生成する
  ⇒　生成した認証情報をダウンロードして、git cloneコマンドを実行する際の認証情報を入力する
  #### 3.作成したリポジトリをローカルにgit cloneする
  コマンド例
  ```
  git clone https://git-codecommit.us-east-1.amazonaws.com/SampleAppRepo C:\path\to\work\directory
  ```
  コマンド実行時には、「2.」の手順でダウンロードした認証情報を使用する 
  #### 4.Laravelの新規プロジェクトをルート直下に作成する
  ```
  composer create-project laravel/laravel sample-app
  ```
  #### 5.gitにすべてのファイルをアップロードする
  ```
  git add -A
  ```
  #### 6.
