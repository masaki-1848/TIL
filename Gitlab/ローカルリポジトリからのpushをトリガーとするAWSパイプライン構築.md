# ローカルのリポジトリからソースをpushして、AWSのCodepipelineを起動し、DockerコンテナをEC2インスタンスに起動する

## ■ アーキテクチャの概要

## ■ アーキテクチャの構築手順

### 1.AWSのElastic Container Registoryからリポジトリを作成する
    ■ 操作手順
    1. サービスから「Elastic Container Registry」を選択する
    2. サイドメニューの「リポジトリ」から「リポジトリを作成」をクリックする
    3. [一般設定]のセクションにて「プライベート」を選択し、リポジトリ名を「gitlab-pipeline-repo」とする
    4. リポジトリを作成する
    5. [リポジトリ名]と[URI]は控えておく

### 2.Pipeline実行用IAMユーザを作成する
    ■ 操作手順
    1. サービスから「IAM」を選択する
    2. サイドメニューの「ユーザー」から「ユーザーの作成」をクリックする
    3. [ユーザー名]を「gitlab-pipeline-act-user」として、次へ
    4. [許可のオプション]は「ポリシーを直接アタッチする」を選択し、下記のロールを追加する
        - AmazonElasticContainerRegistryPublicFullAccess
        - AWSCodePipeline_FullAccess
        - AmazonEC2RoleforAWSCodeDeploy
        - AWSCodeDeployRoleForECS
    5. ユーザを作成する
    6. 作成したユーザの詳細画面より[セキュリティ認証情報]タブを選択し、「アクセスキーを作成」をクリックする
    7. [ユースケース]から「コマンドラインインターフェイス (CLI)」を選択して次へ
        ※AWSのCLIを用いてログイン認証を行うため
    8. [説明タグ値]に「access key for gitlab pipeline」を入力してアクセスキーを作成する
    9. CSVファイルをダウンロードしておく
    10. 再度、ユーザの詳細画面より[セキュリティ認証情報タブ]を選択し、「」
    

### 3.GitlabのCI/CD変数を定義する
    ■定義する変数一覧
    - ECR_REPOSITORY: イメージをプッシュする ECR リポジトリ
    - AWS_ACCESS_KEY_ID: AWS認証アクセスキーID
    - AWS_SECRET_ACCESS_KEY: AWS 認証シークレットアクセスキー
    - AWS_DEFAULT_REGION: AWS デフォルトリージョン  
    ※手順1.で作成したIAMユーザーのアクセスキーとシークレットIDを使用する

    ■ Gitlabの操作手順
    1. Gitlabにアクセスし該当のプロジェクトを開く
    2. サイドメニューの「設定」より「CI/CD」を選択する
    3. [Variables]の「展開」を選択し、上記の変数を設定する

### 4.Gitlab Runnerの定義ファイルを作成する



### 4.AWSのCodePipelineを構築する
#### 4-1.

### 5.ローカルリポジトリからpushしてみる

### 5.まとめ

## ■ 残タスク
- AWS側でDockerコンテナを起動した後に自動でテストを行う処理を追加する
- CI/CDにエラーが発生した場合にpushユーザに通知する機能を追加する
- GitHubで同様のpipelineを構築する
