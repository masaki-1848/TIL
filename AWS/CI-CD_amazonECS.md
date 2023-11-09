# AWS CI/CD for Amazon ECS ハンズオンを試してみる
  ## 参考サイト
  [https://qiita.com/mksamba/items/ffbdbd0a8ca25c3e060e](https://pages.awscloud.com/rs/112-TZM-766/images/AWS_CICD_ECS_Handson.pdf)

  ## 目的
  Code系のサービス（CodeCommit、CodeDeploy、CodePipeline）を用いて、EC2インスタンスにローカルリポジトリからの  
  pushをトリガーとして自動デプロイする仕組みを構築することができたので、次は、Dockerイメージを自動的に登録し  
  Dockerイメージからコンテナを自動的にデプロイする仕組みを作りたいと思い技術を検証するために実施

  ## 今回挑戦するアーキテクチャ


  ## 操作手順
  
  ### 1.ECS環境の構築
  #### 【VPCを作成する】
  CloudFormationを使用し、VPC、InternetGateway、NATgateway、subnet、routetableは一括で作成する  
  ##### 【CloudFormation実行手順】
  1. AWSのコンソールからCloudFormationを開く
  2. [スタックの作成]を選択する
  3. [テンプレートの準備]より「テンプレートの準備完了」を選択し、「テンプレートファイルのアップロード」を選択して任意のファイルをアップロードする
     ※cloudFormationを実行するためのymlファイルには、主に下記項目について記載する
     - AWSTemplateFormatVersion
     - Description
     - Mappings
     - Parameters
     - Resources


     > 注意点
     - Windowsのメモ帳アプリでyamlファイルを作成すると改行コードが「CR（キャリッジリターン）+LF（ラインフィード）」  
     　となるため、CloudFormationに取り込むとエラーが発生する（Linux形式（LF）にする必要がある！！）
     - スカラー値（key:value）と（key: value）では意味が異なる！
       
     　きれいなCloudFormationファイルの書き方は下記のサイトを参考にする
     > https://qiita.com/yes_dog/items/1d35b93278cc1f93d3c4
  4. [スタック名]を「vpc-for-docker-cicd」と入力する
  5. スタック作成をクリックする

  > ポイント！  
  各ResourceのTagsプロパティに指定する「KeyとValue」記載する際に「ハイフン（-）」をValueの前に記載したり、  
  Valueではなく”Values”などと記載するとエラーになる


  #### 【Application Load Valancer（ALB）を作成する】
  ##### 【操作手順】
  1. VPCのダッシュボードからセキュリティグループを選択する
  2. セキュリティグループを作成をクリックする
  3. [セキュリティグループ名]を「docker-app-sg-01」、[説明]に「security group for docker app created by cloudformation」と入力
  4. [VPC]からCloudFormationで作成したVPC「docker-app-vpc」を選択
  5. インバウンドルールとして「HTTP」を「0.0.0.0/0のanywhere IPv4」で設定
  6. 作成する
  7. サービスから「EC2」を選択
  8. サイドメニューから「ロードバランサー」を選択
  9. ロードバランサーの作成をクリック
  10. 「Application Load Balancer 」を作成
  11. [ロードバランサー名]を「docker-app-lb-01」と入力
  12. [ネットワークマッピング]のセクションにて、VPCは「dokcer-app-vpc」を、availabilityzoneを２つともチェックして「publicsubnet1,2」を選択
  13. [セキュリティグループ]として先程作成した「dokcer-app-sg-01」とdefaultを選択
  14. [ターゲットグループ]は新規作成する（※[dummy]という名称で作成）
  15. ロードバランサーを作成する
  16. ALBのリスナータブを開き、defaultの設定を削除する
  17. サイドメニューから「ターゲットグループ」を開き、先ほど作成した「dummy」を削除する

  #### 【ECR（Elastic Container Registry）のリポジトリを作成する】
  ##### 【操作手順】
  1. [Elastic Container Registry]のサービスを開く
  2. サイドメニューから[リポジトリ]を選択し、リポジトリを作成をクリックする
  3. リポジトリの種類は「private」とし、[リポジトリ名]は「docker-app-repo」とする
  4. リポジトリを作成する
  5. 「view push commands」から各種コマンドをメモしておく

  #### 【Cloud9環境の構築】
  ##### 【操作手順】
  1. [Cloud9]のサービスを開く
  2. サイドメニューから「自分の環境」をクリックし、「環境を作成」を選択する
  3. [名前]を「docker-app-cloud」と入力する
  4. [説明]を「cloud9 for docker app」と入力する
  5. [環境タイプ]として「新しいEC2インスタンス」を選択し、インスタンスタイプは「t3.small」を選択する
  6. [プラットフォーム]は「Amazon Linux 2」とする
  7. [タイムアウト]は「30分」を設定しておく
  8. [ネットワーク設定]セクションから「SSM」を選択し、VPCは「docker-app-vpc」を選択する  
    ※「SSH」を使用する場合は、予めSSH接続が可能なインスタンスを立ち上げて置き、環境タイプを選択する際に「既存のインスタンス」を選択する
  9. [サブネット]からは「publicsubent1 or publicsubnet2」を選択する
  10. 作成をクリックする

  > ポイント
  - CloudFormationでサブネットを作成すると、デフォルトでは「IPv4」のIPアドレスの自動割り当てが無効になっているため、  
    Cloud9作成時にEC2インスタンスに接続できずエラーとなる！

  #### 【Git環境の設定】
  1. Cloud9サービス上でコンソール画面を開く
  2. 下記のコマンドを実行する
  ```
  $ git config --global user.name ”yourname”
  ```
  ```
  $ git config --global user.email yourname@abc.com
  ```
  ```
  $ git config --global credential.helper '!aws codecommit credential-helper $@'
  ```
  ```
  & git config --global credential.UseHttpPath true
  ```
  3. Cloud9のIDEを開く
  4. Dockerfileを作成する
  ```txt:Dockerfile
  FROM php:8.1-apache
  COPY src/ /var/www/html/
  ```
  5. [docker-app-cloud-01]フォルダ直下に「src」ディレクトリと「index.php」ファイルを作成する
  ```php:index.php
  <!DOCTYPE html>
  <html lang="ja">
  <head>
    <title>PHP Sample</title>
  </head>
  <body>
    <?php echo gethostname(); ?>
  </body>
  </html>
  ```

  #### 【Dockerイメージを作成する】
  ##### 【操作手順】
  1. Cloud9のサービスを開く
  2. AWSのコンソールを開く
  3. dockerイメージを作成する下記のコマンドを実行する
  ```
  $ docker build -t repositry_name .
  ```
  4. Dockerコンテナを起動する
  ```
  $ docker run --rm -p 8080:80 -d repositry_name:latest
  ```
  5. cloud9 IDEの[Preview]タブから「Preview Running Application」を選択する
  6. index.phpの内容が表示されることを確認する
  7. docker containerを停止する
  ```
  $ docker stop container_id
  ```
  
  #### 【ECRにDockerイメージをpushする】
  ##### 【操作手順】
  1. Cloud9 IDEのコンソールからECRにログインする
  > 【ECR（Elastic Container Registry）のリポジトリを作成する】の手順５でメモしたコマンドを使用すること！
  - ログイン
  ```
  aws ecr get-login-password --region ～～
  ```
  ※「Login Succeeded」が表示されることを確認する
  - タグ付け
  ```
  docker tag repositry_name:latest ○○.dkr.ecr.region_name.amazonaws.com/repository_name:latest
  ```

  - イメージのpush
  ```
  docker push ○○.dkr.ecr.region_name.amazonaws.com/repositry_name:latest
  ```
  2. サービスから[Elastic Container Registry]を開く
  3. サイドメニューから「repositry」を選択し、先ほどpushしたイメージが存在することを確認する

  ### 2.AWS Fargate環境の構築
  #### 【IAMロールの設定】
  ##### 【操作手順】
  1. サービスから「IAM」を選択する
  2. サイドメニューから「ロール」を選択し、ロールを作成をクリックする
  3. [信頼されたエンティティタイプ]から「AWSのサービス」を選択
  4. [ユースケース]から「CodeDeploy」、「CodeDeploy ECS」を選択して次へを選択する
  5. [ポリシー名]に「AWSCodeDeployRoleForECS」ポリシーが表⽰されていることを確認し次へ
  6. [ロール名]に「CodeDeployRoleForECS」を入力してロールを作成する

  #### 【Fargateクラスターを作成する】
  ##### 【操作手順】
  1. サービスから「Elastic Container Service」を選択する
  2. サイドメニューから「クラスター」を選択し、クラスターの作成をクリックする
  3. [クラスター名]に「fargate-cluster」を入力する
  4. [インフラストラクチャ]として「AWS Fargate (サーバーレス)」を選択し、作成する
  5. サイドメニューから「タスク定義」を選択し、「新しいタスク定義の作成」をクリックする
  6. [タスク定義ファミリー]の名称を「docker-sample-fargate」とする
  7. [タスクロール]は未選択とする
  8. メモリとCPUは「0.5GB/0.25vCPU」とする
  9. コンテナセクションの[名前]は「docker-sample-fargate」とし、リポジトリURIは作成したリポジトリのURLを指定する
  10. コンテナのメモリは[ソフト]が「0.128」、[ハード]が「0.256」で設定を行う

  #### 【Fargateのサービスを作成する】
  ##### 【操作手順】
  1. サービスから「Elastic Container Service」を選択する
  2. サイドメニューから「タスクの定義」をクリックする
  3. 先程作成したタスク定義を選択する
  4. [デプロイ]から「サービスの作成」をクリックする
  5. [既存のクラスター]から「fargate-cluster」を選択する
  6. [サービス名]は「docker-fargate」とする
  7. [ネットワーキング]のセクションからVPCは「docker-app-vpc」、サブネットは「private-subnet1,2」、セキュリティグループは「default」を選択する
  8. [ロードバランシング]のセクションから「Application Load Valancer」、「既存のロードバランサー」、「docker-app-lb-01」を選択する
  9. [ターゲットグループ]を新規作成し、名称を「docker-app-tgt-group1」「docker-app-tgt-group2」とする
  10. [ヘルスチェックパス]を「/index.php」とする
  ```
  デプロイの設定を「Blue/Green」にしないと、パイプラインを作成する際にエラーが発生するので注意すること！！
  ```
  11. 作成をクリックする
  12. サービスから「EC2」を選択する
  13. サイドメニューから「ターゲットグループ」を選択する
  14. 先程作成したターゲットグループ「docker-app-tgt-group」を選択し、「ターゲット」タブに表示されているタスクがHealthyになっていることを確認する
  15. ALBのエンドポイントにアクセスし、アプリケーションのページが表示されることを確認する
      EC2のサービスから、サイドメニューの「ロードバランサー」を選択し、詳細画面の「DNS名」を使用する

  #### 【CI/CDパイプラインを実現する】
  ##### 【使用するAWSサービス】
  1. CodeCommit
  - ソースコードのバージョン管理に使用する
  ```
  - セキュアかつスケーラブルなGit互換のリポジトリサービス
  - スタンダードなGit Toolからアクセスが可能
  - PushなどのイベントをトリガーにSNS/Lambdaを呼び出し可能
  ```
  2. CodeBuild
  - ビルドの自動化に用いる
  ```
  - スケーラビリティに優れたビルドサービス
  - ソースのコンパイル、テスト、パッケージ生成をサポート
  - Dockerイメージの作成も可能
  ```
  3. CodeDeploy
  - デプロイの自動化に用いる
  ```
  - S3またはGitHub上のコードをあらゆるインスタンスにデプロイする
  - デプロイを安全に実行するための様々な機能を提供している
  - In-Place（ローリング）およびBlue/Greenのデプロイをサポート
  ```
  4. CodePipeline
  - ワークフローを管理する
  ```
  - リリースプロセスのモデル化と見える化を実現
  - カスタムアクションによる柔軟なパイプラインの生成が可能
  - 様々なAWSサービスや3rdパーティ製品との統合をサポート
  ```

  ### 3.AWS Code Servicesを利用したCI/CDパイプラインの構築
  #### 【パイプラインの概要】
  1. ローカルで各developerがgit pushしてリポジトリを更新
  2. CodeCommitへのpushをCodePipelineが検出し、パイプラインを開始
  3. CodeBuildがDockerイメージをビルドし、Elastic Container Registryへプッシュ
  4. CodeDeployがElastic Container RegistryのDockerイメージをElastic Container ServicesクラスタにBlue/Greenデプロイメント

  #### 【操作手順】
  ##### 【CodeCommitのリポジトリをクローンする】
  1. サービスから「CodeCommit」を選択する
  2. サイドメニューのリポジトリを選択し、「リポジトリの作成」をクリックする
  3. [リポジトリ名]を「docker-cicd-repo」とする
  4. [説明]に「repositry for docker ci/cd structure 」と記載する
  5. 作成する
  6. サービスから「Cloud9」を選択する
  7. 作成したCloud9の環境を選択し、IDEを起動する
  8. ターミナルにhttpsによるgit cloneコマンドを実行する
  ```
  $ git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/repositry_name
  ```
  9. リポジトリフォルダ直下に下記のファイルを格納する
  - Dockerfile
  - buildspec.yml (※1)
  - appspec.yml
  - taskdef.json (※2)
  - src/index.php  
  ※1 buildspec.ymlのpre_buildセクションに記載するコマンドは【ECR（Elastic Container Registry）のリポジトリを作成する】の手順５で取得したコマンドを使用すること  
  ※2 taskdef.jsonは、サービスから「ECS」を選択し、サイドメニューの[タスク定義]からFargateを選択し、最新のRivisionに表示されているJsonタブの内容をコピーする
  

  10. リポジトリにpushする
  ```
  $ cd repositry_name
  $ git add -A
  $ git commit -m "my first commit"
  $ git push origin master
  ```
  11. CodeCommitから該当のリポジトリにソースがpushされていることを確認する

  ##### 【アプリケーションを作成する】
  1. サービスから「CodeDeploy」を選択する
  2. サイドメニューの「デプロイ > アプリケーション」を選択し、アプリケーションの作成をクリックする
  3. [アプリケーション名]は「AppECS-fargate-cluster-docker-sample-fargate」とする
  4. [コンピューティングプラットフォーム]は「Amazon ECS」とする
  5. アプリケーションを作成する
  6. デプロイグループの作成をクリックする
  7. [デプロイグループ名]を「DgpECS-fargate-docker-sample-fargate」とする
  8. [サービスロール]は「CodeDeployRoleForECS」とする
  9. [ECSのクラスター名]は「fargate-cluster」とする
  10. [ECSのサービス名]は「docker-fargate」とする
  11. [Load Balancer]は、「docker-app-lb-01」とする

  ##### 【パイプライン作成】
  1. サービスから「CodePipeline」を選択する
  2. サイドメニューの「パイプライン」を選択し、パイプラインの作成をクリックする
  3. [パイプライン名]は「docker-app-pipeline」とする
  4. [サービスロール]は新規作成を選択し、次へを選択する
  5. [ソースプロバイダー]から「CodeCommit」、リポジトリは「docker-cicd-repo」、ブランチは「master」を選択し、次へ
  6. ビルドステージの[プロバイダーを構築する]から、「CodeBuild」を選択
  7. [リージョン]は作業中のリージョンを選択し、プロジェクトを作成するをクリックする
  8. [プロジェクト名]を「docker-cicd-build」とする
  9. 環境のセクションは下記の設定とする
  - [環境イメージ] => [マネージド型イメージ]
  - [コンピューティング] => [EC2]
  - [オペレーティングシステム] => [Ubuntu]
  - [ランタイム] => [Standard]
  - [イメージ] => [aws/codebuild/standard:7.0]
  - [イメージのバージョン] => [最新のイメージ]
  - [特権付与] => [有効]
  - [ロール名] => [codebuild-build_name-service-role]
  10. CodePipelineに進むを選択する
  11. デプロイステージにて[デプロイプロバイダー]より「Amazon ECS(Blue/Green)」を選択する
  12. [クラスター名]は「fargate-cluster」を選択
  13. [リージョン]は作業用のリージョンを選択
  14. [AWS CodeDeploy アプリケーション名]は、ECSでサービスを登録すると自動的に生成されるアプリケーション名を指定する
  15. [AWS CodeDeploy デプロイグループ]も同様
  16. [Amazon ECS タスク定義]と[AWS CodeDeploy AppSpec ファイル]は「BuildArtifact」を選択する
  17. パイプラインを作成する

  #### 【CodeBuildのIAMロールを編集する】
  1. サービスから「IAM」を選択する
  2. サイドメニューの「ロール」を選択し、検索欄に「CodeBuild」を入力して、表示される「codebuild-build_name-service-role」を選択する
  3. [許可]タブの「許可を追加」メニューから「ポリシーのアタッチ」を選択する
  4. 検索欄に「Container」を入力し、「AmazonEC2ContainerRegistryPowerUser」をアタッチする

  #### 【CondePipelineの設定編集】
  1. サービスから「CodePipeline」を開く
  2. 先程作成した「docker-app-pipeline」を選択する
  3. 「編集する」ボタンをクリックする
  4. 「Deploy」ステージを編集する
  5. [入力アーティファクト]の「追加」をクリックし、「SoucrArtifact」を追加する
  6. [Amazon ECS タスク定義]と[AWS CodeDeploy AppSpec ファイル]を「SourceArtifact」に変更する
  7. [タスク定義の動的な更新イメージ - オプショナル]の「入力アーティファクトを持つイメージの詳細」を「BuildArticact」、「タスク定義のプレースホルダー文字」を「IMAGE1_NAME」とする
  8. 完了ボタンをクリックする
  9. サービスから「CodeDeploy」を開く
  10. サイドメニューの「アプリケーション」を開き、「AppECS-fargate-cluster-docker-fargate-01」を選択して、デプロイグループを編集する
  11. [デプロイ設定]セクションの時間を「0」分を「5」に変更する
  12. パイプラインを実行する
  13. pipelineの動作を検証する

  #### 【余談1：Blue/GreenとIn-Placeデプロイメントについて】
  - Blue/Green
  > 現状の本番環境（Blue）とは別に新しい本番環境（Green）を構築したうえで、ロードバランサーの接続先を切り替えるなどを行い、  
    新しい本番環境をリリースするデプロイメント方法

  - In-Place
  > 現在稼働中のサービス実行環境に対し、アプリケーションのみを入れ替えるデプロイ方法
    一定台数ずつ順番にデプロイしている方式をローリングという。

  #### 【余談2:Pipelineのエラー（CodeBuild）】
  - 基本的に原因は「buildspec.yml」ファイルの記述内容に問題がある
  - CodeBuildでdefaultで利用可能な環境変数は下記のサイトを参照
  https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/build-env-ref-env-vars.html
  暗号化させたい変数はSystems managerで管理する方法もありかと

  #### 【余談3:Pipelineのエラー（CodeDeploy）】
  - taskdef.ymlのキーの「tags:[]」だとエラーになるので、tagsそのものを削除してしまう
  - 「appspec.ymlファイルが存在しない」と怒られるときは、「buildspec.yml」のartifactsセクションに「- appspec.yml」を追記すると解消する
  - テンプレートがパースできないというエラーメッセージが出るときは、「appspec.yml」の構文が誤っている可能性が高い

  ### 4. 後片付け
  #### 【操作手順】
  1. CodePipelineを削除する
  2. CodeDeployを削除する
  3. CodeBuildを削除する
  4. CodeCommitを削除する
  5. S3 Bucketを削除する
  6. ECRを削除する
  7. ECS タスク定義を削除する
  8. ECS クラスターを削除する
  9. ALBを削除する
  10. ALBターゲットグループを削除する
  11. Cloud9を削除する
  12. CloudFormationを削除する
  13. IAMロールを削除する
  ```
  対象は４種類
  - cwe-role-us-east-1-docker-app-pipeline
  - AWSCodePipelineServiceRole-us-east-1-docker-app-pipeline
  - CodeDeployRoleforECS
  - codebuild-docker-app-build-service-role
  ```
  14. CloudWatch Logsを削除する
