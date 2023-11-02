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
  1. すべてのファイルをステージング
  ```
  git add -A
  ```
  2. ファイルをコミットする
  ```
  git commit -m "Add sample application files"
  ```
  3. CodeCommitのリポジトリにローカルリポジトリからプッシュする
  ```
  git push
  ```
  ※上記を実行すると、CodeCommitのターゲットリポジトリの「main」ブランチにファイルが追加される
  
  #### 6.
