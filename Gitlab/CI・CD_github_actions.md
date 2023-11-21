# GiutHub Actionsを用いたCI/CDアーキテクチャの構築
## 【目的・背景】
  GitHubやGitLab、AWSのCodeCommitなど、ソース管理のサービスが多数存在する中で、
  これまでに、AWSのCode系サービスと連携したCI/CDのアーキテクチャの技術検証をいくつか行ってきた。

  AWSのCode系サービスを用いたCI/CDアーキテクチャは簡単に構築できるメリットがある一方で、
  コスト面の問題があるため、無料利用枠が多く、単位コストの安いGitHub Actionsを用いた
  CI/CDアーキテクチャの構築をすることにした。

  ■ コスト
    - Freeプラン　：　月2000分まで無料。500MBストレージ利用可能。
    
## 【GitHub Actionsに関する用語】
  1. ワークフロー
    - 任意のプロジェクトをビルドしたり、テストしたり、デプロイしたりなどの様々な作業のまとまり
    - プロジェクトのルートに.github/workflowsというディレクトリを作り、その中に*.ymlファイルを作り、具体的な処理を記述する
    - *.ymlファイルが１つのワークフローとして定義され、プロジェクトで複数のワークフローを定義できる  
  2. ジョブ
    - ワークフローの中で実行される処理のかたまりのこと
    - ワークフローに複数定義することが可能
    - ジョブは並行して処理される
  3. ステップ
     - ジョブの中で実行される一連のタスク
     - ワークフローを校正する様々なタスクの最小単位
  4. アクション
     - ステップとして実行されるタスクの中で再利用可能なコードの単位のこと

## 【GitHub ActionsからECSにアプリケーションをデプロイする】
### 1. アーキテクチャの全体図
  - LocalPC ⇒ GutHubレポジトリにソースをpush

### 2. AWSのOIDC(OpenID Connect)を設定する
  ■ 概要
  - OIDCを設定すると、AWSの認証情報をGitHub上のシークレットに保存する必要がなくなり、
    GitHub ActionsのワークフローがAWSのリソースにアクセスできるようになる
  - 仕組みとしては、サードパーティの外部アプリケーションからAWSに何らかのジョブを依頼する際に、
    依頼してきたアプリケーション（プロバイダー）のIDを予め登録しておくことで、認証を行い、
    問題が無ければJWT（Json Web Token）を発行して、JWTの記載内容に基づいて許可ポリシーを確認する
  - サードパーティの外部アプリケーションからすると、大事な情報（AWSの認証情報）を管理するというリスク
    から解放されるというメリットがある

  ■ 設定手順
  - AWSにアクセスし、サービスから「IAM」を選択する
  - サイドメニューの[アクセス管理 > IDプロバイダ]を選択し、[プロバイダを追加]をクリックする
  - [プロバイダのタイプ]から「OpenID Connect」を選択しする
  - [プロバイダのURL]は、GitHubの場合は「https://token.actions.githubusercontent.com」にする
  - [サムプリント取得]をクリックする
  - [対象者]には「sts.amazonaws.com」を入力する
  - [プロバイダを追加]をクリックする
    
### 3. GitHub Actionsの設定
  1. 作業用のpublicリポジトリを新規作成する
     - 名称は「github_actions_repo」とする
  3. 作成したリポジトリに移動し、「Actions」タブをクリックする
  4. 今回はECSにデプロイするアーキテクチャを作るので「Deploy to Amazon ECS」を選択する
  5. リポジトリのルートディレクトリに「.github/workflows/aws.yml」が作成される
  6. 要件に合わせてymlファイルを編集する（今回は一旦省略してまた別の記事で作成する）
  7. commit changesを押下する

### 4. git cloneしてローカルでファイルを編集する

  1. 以下のコマンドでリポジトリをクローンする
  ```
  git clone https://github.com/account_name/repository_name.git
  ```
  2. ymlファイルを編集  
  - AWS認証(56行目～61行目)
     ```
     - name: Configure AWS credentials
       uses: aws-actions/configure-aws-credentials@v1
       with:
         aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
         aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
         aws-region: ${{ env.AWS_REGION }}
     ```
     こちらの書き方の場合、ymlファイルにAWSのクレデンシャル情報を直接記載することになり、
     セキュリティ的にNGなので、一時的なトークンを GitHub Actions が受け取り、それを使って CICD を行うパイプライン構築を行う
     https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
  - ワークフロー内でOpenID Connect を使用して、AWSで認証を行う
  2-1. OpenID Connect (OIDC) ID プロバイダーの作成
  - AWSにアクセスする
  - サービスからIAMを選択する
  - サイドメニューから「アクセス管理 > IDプロバイダ」を選択する
  - [プロバイダを追加]をクリックする
  - [プロバイダのタイプ]から「OpenID Connect」を選択する
  - [プロバイダの URL]には「https://token.actions.githubusercontent.com」を指定する
  - [対象者]には「sts.amazonaws.com」を指定する
    ※対象者とは、OIDC の audience（aud クレームの値）の事で、クライアントIDを意味する。
      この値は GitHub Actions で CICD を実行するにあたってaws-actions/configure-aws-credentialsという
      GitHub Actions 公式のアクションを利用するが、そのアクション自体がクライアントになり、
      そのため audience 名としては sts.amazonaws.com になる
  - [プロバイダを追加]をクリックする
  2-2. プロバイダーのロールを設定する
  - AWSにアクセスし、サービスから「IAM」を選択する
  - サイドメニューから「ロール」を選択し、「ロールを作成」をクリックする
  - [信頼されたエンティティタイプ]より「カスタム信頼ポリシー」を選択する
    ■ カスタム信頼ポリシーへの記載内容
    ```
     {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Sid": "",
                  "Effect": "Allow",
                  "Principal": {
                      "Federated": "arn:aws:iam::<AWSアカウントID>:oidc-provider/token.actions.githubusercontent.com"
                  },
                  "Action": "sts:AssumeRoleWithWebIdentity",
                  "Condition": {
                      "StringEquals": {
                          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                          "token.actions.githubusercontent.com:sub": "repo:<GitHubユーザー名>/<GitHubリポジトリ名>:ref:refs/heads/<ブランチ名>"
                      }
                  }
              }
          ]
      }
     ```
  - [次へ]をクリックする
  - [ポリシーを許可]からgithub actionsから呼び出すAWSサービスの実行権限を持たせるポリシーを追加し、「次へ」をクリックする
    - AmazonElasticContainerRegistryPublicFullAccess
    - AmazonECSTaskExecutionRolePolicy
    - AmazonS3ReadOnlyAccess
  - [ロール名]に「githubActionsExeRole」を入力する
  - [説明]に「This role is execution admin for deployment ecr triggered by github actions.」を入力する
  - [ロールを作成]をクリックする
  2-3. 【インラインポリシーから設定する場合】
    ```
     {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Action": "ecr:GetAuthorizationToken",
                  "Effect": "Allow",
                  "Resource": "*"
              },
              {
                  "Action": [
                      "ecr:UploadLayerPart",
                      "ecr:PutImage",
                      "ecr:InitiateLayerUpload",
                      "ecr:CompleteLayerUpload",
                      "ecr:BatchCheckLayerAvailability"
                  ],
                  "Effect": "Allow",
                  "Resource": "<ECRリポジトリのARN>"   （※１）
              }
          ]
      }
    ```
    ※１：サービスの「Elastic Container repository」のリポジトリから該当のリポジトリを選択して概要から確認できる

  3. ECRのリポジトリを作成する
     - AWSにアクセスし、サービスから「Elastic Container Registry」を選択する
     - サイドメニューの[リポジトリ]を選択し、「リポジトリを作成」をクリックする
     - [可視性設定]から「プライベート」を選択する
     - [リポジトリ名]を「github_actions_repo」とする
     - [タグのイミュータビリティ]を「有効」にする（※同一タグ名（latestなど）で運用できないようにする）
     - [リポジトリを作成]をクリックする

  4. Dockerfileを作成する
     - 今回は検証のために、シンプルなphp8.1とapacheがインストールされている公式イメージ（php:8.1-apache）を使用する
       ```
       FROM php:8.1-apache
       COPY src/ /var/www/html/
       ```

  5. コンテナイメージを初回プッシュする
     - AWSにアクセスし、サービスから[cloud9]を選択する
     - [環境を作成]をクリックする
     - [名前]は「ecr-first-upload-env」とする
     - [環境タイプ]は「新しいEC2インスタンス」を選択する
     - [インスタンスタイプ]は「t2.micro (1 GiB RAM + 1 vCPU)」とする
     - [プラットフォーム]は「Amazon Linux 2」とする
     - [ネットワーク設定]の「接続」は「SSM」を選択する
     - [作成]をクリックする
     - [IDEを開く]をクリックする
     - [ECR]のリポジトリを選択し、[プッシュコマンドを表示]から１つ目のコマンドを実行してDockerにログインする
     - ドキュメントルートに[Dockerfile]と[src/index.php]ファイルをCloud9上で作成する
     - [ECR]のリポジトリを選択し、[プッシュコマンドを表示]から２つ目のコマンドを実行してDockerイメージをビルドする
     - [ECR]のリポジトリを選択し、[プッシュコマンドを表示]から３つ目のコマンドを実行してリポジトリにイメージをプッシュできるようにタグ付けする
     - [ECR]のリポジトリを選択し、[プッシュコマンドを表示]から４つ目のコマンドを実行してリポジトリにイメージをプッシュする

     ※予めコンテナイメージを「latest」タグでプッシュしておかないと、ECSのクラスターおよびサービスを作成したときに
     　CloudFormationのタスクでエラーが発生する
     
  6. github Actionsのワークフローを作成する
     - githubにアクセスし、作業用のリポジトリを選択して「actions」タブより
       [new workflow]を選択する
     - [Deploy to Amazon ECS]の「configure]をクリックする
       
     ■ ポイント
     - OIDC を使用した AWS 認証には aws-actions/configure-aws-credentials アクションを使用する
     - ECRへのログインには aws-actions/amazon-ecr-login アクションを使用する
     
     ■ 全体のサンプルワークフロー
     ```
      name: github-actions-ecr-cicd

      # ワークフローのトリガーを「master」ブランチへのpushとpull requestとする
      on:
        push:
          branches: [ master ]
        pull_request:
          branches: [ master ]
      
      # 実行するジョブの定義
      jobs:
        deploy:
          name: Deploy
          runs-on: ubuntu-latest
          environment: production
      
          # permisssionを設定するとOIDCが使用可能となる
          permissions:
            id-token: write
            contents: read
      
          steps:
          - name: Checkout
            uses: actions/checkout@v4
      
          # AWS認証
          - name: Configure AWS credentials
            uses: aws-actions/configure-aws-credentials@master 
            with:
              # GitHubのリポジトリ上でsecretsを設定してクレデンシャルを管理する
              aws-region: ${{ secrets.AWS_REGION }}
              role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}::role/githubActionsExeRole
      
          # ECRにログイン
          - name: Login to Amazon ECR
            # outputsを参照するためのidを設定（？）
            id: login-ecr
            uses: aws-actions/amazon-ecr-login@v1
      
          # Dockerイメージをbuild & pushする
          - name: Build, tag, and push image to Amazon ECR
            id: build-image
            env:
              # ECRのレジストリを'aws-actions/amazon-ecr-login'アクションの`outputs.registry`から取得
              ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
              # イメージをpushするECRリポジトリ名（クレデンシャル同様に秘匿管理するべき？）
              ECR_REPOSITORY: github_actions_repo
              # 任意のイメージタグ（今回はGitのコミットハッシュにしておく）
              IMAGE_TAG: ${{ github.sha }}
            run: |
              # Build a docker container and push it to ECR so that it can be deployed to ECS.
              docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
              docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
              echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

          # AWSのECSのタスク定義はdockerでいうところのdocker-compose.ymlに当たる
          # 取得するイメージや、コンテナ起動時の要件などを定義する
          # IAMのロールとして「AmazonECSTaskExecutionRolePolicy」が付与されていれば実行可能となる
          # タスク定義の更新処理を行う
          - name: Fill in the new image ID in the Amazon ECS task definition
            id: task-def
            env:
              # 任意のコンテナ名を定義する
              CONTAINER_NAME: docker-app
              # ECSのタスク定義ファイルは「application_root_path/aws/taskdef.json」ファイルを参照する
              ECS_TASK_DEFINITION: ./aws/taskdef.json
            uses: aws-actions/amazon-ecs-render-task-definition@v1
            with:
              task-definition: ${{ env.ECS_TASK_DEFINITION }}
              container-name: ${{ env.CONTAINER_NAME }}
              # イメージ名はECRのoutputから取得する
              image: ${{ steps.build-image.outputs.image }}
      
          - name: Deploy Amazon ECS task definition
            uses: aws-actions/amazon-ecs-deploy-task-definition@v1
            with:
              task-definition: ${{ steps.task-def.outputs.task-definition }}
              service: ${{ env.ECS_SERVICE }}
              cluster: ${{ env.ECS_CLUSTER }}
              wait-for-service-stability: true
     ```
     ■ ポイント
       - 秘匿化したい情報（AWSのクレデンシャルやECRのリポジトリ名）等、
         に関しては、GitHubのSecretsに登録しておく
         - GitHubにアクセスし、作業対象リポジトリを選択して「Settigs」をクリックする
         - サイドメニューの[Secrets and variables]から、「new repository secret」をクリックする
         - 登録対象のキーを作成する
           例）AWS_ACCOUNT_ID, AWS_REGION, ECR_REPOSITORY_NAMEなど
       - 各キーの意味
         - job：起動するジョブを記述するセクション
         - build：jobsのIDを表している
         - strategy：各jobsを実行する際のoptionを指定できる
         - runs-on：github上でホストされているubuntuで処理が実行されることを指す
         - steps：ジョブ内で実際に処理する内容をグルーピングする
         - name：ワークフロー名
         - run：処理したい内容
         - use：コミュニティで定義されたアクションを利用する際に記述
         - uses：actions/checkout@v2は「actions/checkout@v2」という名前のコミュニティアクションのv2を取得するように
           ジョブに指示をするという意味
         - with：アクションによって定義される入力パラメータのmap（key：value形式のデータを持つ)
       - タスク定義ファイル（taskdef.json）はAWSのECSの画面から取得して加工したものを利用する
        
           
  7. ECSのタスク定義ファイル（taskdef.json）を作成する
     - Dockerコンテナをデプロイする方法２種類
       - Fargate
       - - EC2  
     - 事前準備
       - FargateもしくはEC2インスタンスをデプロイするためのVPCとサブネットを準備しておく必要がある
         ⇒　CloudFormationで一括作成してもよいし、手動で設定してもよい
         ⇒　[スタック名]を「github-actions-cicd-env」とする
     - 設定手順
       - クラスターの作成
         1. AWSにアクセスし、サービスから「Elastic Container Service」を選択する
         2. サイドメニューの[クラスター]を選択する
         3. [クラスターの作成]をクリックする
         4. [クラスター名]は[github-actions-cicd-cluster]とする
         5. [インフラストラクチャ]は[AWS Fargate (サーバーレス)]とする
         6. [作成]をクリックする
       - タスク定義の作成
         1. AWSにアクセスし、サービスから「Elastic Container Service」を選択する
         2. サイドメニューの[タスク定義]を選択し、「新しい託す定義の作成」をクリックする
         3. [タスク定義ファミリー]は[github-actions-cicd-fargate]とする
         4. [インフラストラクチャの要件]は「Amazon Fargate」を選択し、[タスクサイズ]から[CPU]は「0.5vCpu」、[メモリ]は「1GB」とする
       　5. [コンテナの名称]は「php-apache」とし、[イメージURI]には[ECR]のリポジトリから「プッシュの表示」で確認できるイメージとタグ名を指定する
         6. [リソース割り当て制限]より「メモリのハード制限」と「メモリのソフト制限」は「1GB」としておく
            ※ハード制限：メモリ使用量がこの値を超えると強制的にコンテナが終了する
            ※ソフト制限：コンテナに予約するメモリ量（MiB）
         7. [ログ収集]のチェックは外す（検証用コンテナのため）
         8. [作成]をクリックする
       - サービスを作成する
         1. AWSにアクセスし、サービスから「Elastic Container Service」を選択する
         2. サイドメニューの[クラスター]から「手順1.」で作成したクラスターを選択し、概要画面を開く
         3. [サービス]タブより、「作成」をクリックする
         4. [コンピューティングオプション]から「起動タイプ」を選択する
         5. [起動タイプ]から「Fargate」を選択する
         6. [デプロイ設定]セクションの設定
           - [アプリケーションタイプ]：サービス
           - [クラスターファミリー]：github-actions-cicd-fargate
           - [サービス名]：php-apache-deploy
           - [サービスタイプ]：レプリカ
           - [必要なタスク]：1
         7. [ネットワーキング]セクションを設定する
           - [vpc]：github-actions-cicd-vpc
           - [subnet]：github-actions-cicd Public Subnet
           - [security group]：default、github-cicd-app-sg
           - [パブリックIP]：オン
         8. [作成]をクリックする

  8. GitHubリポジトリのプロジェクトルートディレクトリに、taskdef.jsonファイルを追加する
     7-1. [手順6.]で作成したタスク定義を開き、JSONタブを選択する
     7-2. [JSON]の全文をコピーする
     7-3. エディター等で新規ファイルを作成し、7-2でコピーした内容を張り付ける
          ※最下行の”tags:[]”は削除する
     7-4. 名称を「taskdef.json」として保存する
     7-5. ビルドしたイメージを動的にタスク定義に紐づけるため、imageに環境変数を指定する

  9. 試しにgit pushしてみる
     - 


  

  
