# ローカルのリポジトリからソースをpushして、AWSのCodeCommitのリポジトリにミラーリングを行い、pushミラーリングをトリガーとしてCodepipelineを起動し、DockerコンテナをEC2インスタンスに起動する

■ 参考サイト
https://qiita.com/chocomintkusoyaro/items/41a1a5feb0fe080b591e

## ■ アーキテクチャの概要

## ■ アーキテクチャの構築手順

### 1. AWSのCodeCommitにGitlabのリポジトリミラーリング用リポジトリを作成する
    ■ 操作手順
    1. サービスから「CodeCommit」を選択する
    2. サイドメニューから「ソース > リポジトリ」を開き、「リポジトリを作成」をクリックする
    3. [リポジトリ名]を「gitlab-mirror-repo」とする
    4. [説明]に「validate the gitlab CI/CD」と入力する
    5. 作成をクリックする

### 2. AWSのElastic Container Registoryにリポジトリを作成する
    ■ 操作手順
    1. サービスから「Elastic Container Registry」を選択する
    2. サイドメニューの「リポジトリ」から「リポジトリを作成」をクリックする
    3. [一般設定]のセクションにて「プライベート」を選択し、リポジトリ名を「gitlab-pipeline-repo」とする
    4. リポジトリを作成する
    5. [リポジトリ名]と[URI]は控えておく 

### 3. CodeCommitへのアクセス用IAMユーザを作成する
    ■ 操作手順
    1. サービスから「IAM」を選択する
    2. サイドメニューの「ユーザー」から「ユーザーの作成」をクリックする
    3. [ユーザー名]を「gitlab-mirroring-user」とする
    4. [許可オプション]は「ポリシーを直接アタッチする」を選択し、下記のロールを追加する
        - AWSCodeCommitPowerUser
    5. ユーザを作成する
    6. 作成したユーザーの詳細画面より「セキュリティ認証情報」タブを開く
    7. [AWS CodeCommit の HTTPS Git 認証情報]のセクションより「認証情報を生成」をクリックする
    8. 認証情報をダウンロードしておく

### 4. Gitlabのリポジトリからpushしてミラーリングされることを確認する
    ■ 操作手順
    1. 作業用に新たにプライベートリポジトリを作成する
    2. 作成したリポジトリをローカルにクローンする
        - 例：git clone https://○○/repository_name.git
    3. 任意のファイルを作成し、masterブランチにpushする
        - git add -A
        - git commit -m "my first commit"
        - git push
    4. gitlabにアクセスし、[設定 > リポジトリ]を開き、「Mirroring repositories」の「展開」をクリックする
    5. [Git リポジトリURL]に手順1で作成したCodeCommitのリポジトリURLを入力する(※)
        ※URLは直接コピーするのではなく、「https:://」の後にCodeCommit実行権限を付与したIAMユーザのGit HTTPS認証アカウントIDを指定すること
        https://<your_aws_git_userid>@git-codecommit.<aws-region>.amazonaws.com/v1/repos/<your_codecommit_repo>
    6. [Only mirror protected branches]にチェックを入れる
    7. [パスワード]を入力して、「ミラーリポジトリ」をクリックする
    8. 再度ローカルリポジトリから、ファイルを編集し、masterブランチにソースをpushする
    9. AWSのCodeCommitにアクセスし、変更が反映されていることを確認する
        ※保護されたブランチの変更がpushされる場合は即座に反映されるが、保護されたブランチへのmergeをトリガーとしたミラーリングでは多少の遅延が発生する

### 5. AWSのCodeDeployからアプリケーションを作成する
    ■ 操作手順   
    [IAMロールの作成]
    1. サービスから「IAM」を選択する
    2. サイドメニューから「ロール」を選択する
    3. [ロールを作成]をクリックする
    4. [信頼されたエンティティタイプ]から「AWSのサービス」を選択する
    5. [ユースケース]から「CodeDeploy」「CodeDeploy - ECS」を選択する
    6. [ポリシー]に「AWSCodeDeployRoleForECS」が選択されていることを確認し、次へを選択する
    7. [ロール名]を「GitlabMirrorDeployECSRole」と入力し、ロールを作成する
    
    [アプリケーションの作成]
    1. サービスから「CodeDeploy」を選択する
    2. サイドメニューから「デプロイ > アプリケーション」を選択し、「アプリケーションの作成」をクリックする
    3. [アプリケーション名]に「gitlab-mirror-deploy」と入力する
    4. [コンピューティングプラットフォーム]から「Amazon ECS」を選択する
    5. [アプリケーションの作成]をクリックする
    6. [デプロイグループの作成]をクリックする
    7. [デプロイグループ名]に「gitlab-mirror-deploygroup」と入力する
    8. [サービスロール]から先程作成した「GitlabMirrorDeployECSRole」を選択する
    9. []

## ■ 残タスク
- AWS側でDockerコンテナを起動した後に自動でテストを行う処理を追加する
- CI/CDにエラーが発生した場合にpushユーザに通知する機能を追加する
- GitHubで同様のpipelineを構築する
