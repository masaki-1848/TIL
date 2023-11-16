# GitHubのGitHub ActionsとAWSのCode系サービスを利用したCI/CDの仕組みを構築
## 【概要】
  いまさらなアーキテクチャではあるが、GitHubとAWSのCode系サービスを連携して、
  ローカルからリリースブランチへのpushもしくはmergeをトリガーとして、
  Dockerイメージをリポジトリに登録し、リポジトリからイメージを取得してコンテナを起動する
  CI/CDのアーキテクチャを作成したいと思い、今回のまとめ記事を作成することにした。  

  また、githubのCI/CDを考えるときは、AWSのCode系サービスよりも
  github actionsを使用する方がコストが安く、IaCの管理も楽になるとの意見もあるため、
  ２つの方法を検証したいと思う。
  
  今回のような技術検証を日々積み重ねていき、企業内におけるテックリードを目指して、
  地道に努力していきたいと思う。

## パターン1. GitHubとAWS Code系サービスの組み合わせによるCI/CD構築
### ■ GitHubの操作

#### 1. リポジトリを作成する
  今回は検証のために、煩わしい設定を省略するために「パブリック」リポジトリを作成することにした。
  名称は「aws-cicd-repo」としている。

#### 2. ローカルにリポジトリをcloneする
  ローカル環境に作業用フォルダを作成し、下記コマンドで
  リポジトリをcloneした。
  ```
  git clone https://github.com/aws-cicd-repo.git
  ```
  cloneしたレポジトリ名のフォルダが作業用フォルダに追加されていることを確認する

#### 3. 試しにgitコマンドを操作してみる
  作業用フォルダからレポジトリ名のフォルダに移動する
  ```
  cd ./docker-aws-cicd-repo
  ```
  gitのconfig設定を行う
  ```
  git config --global user.name "github_account_user_name"
  git config --global user.email "github_account_user_email"
  ```
  リモートにpushする
  ```
  git add -A
  git commit -m "my first commit"
  git push
  ```
  githubにアクセスし、pushしたファイルがリポジトリに追加されているかを確認する

  【ポイント】
  - もしWindowsでgitのconfig情報を誤って設定してしまった場合は、
    コントロールパネルから一度削除したうえで、ブラウザから認証すると正常にgitコマンドが操作できるようになる

### ■ AWSの操作
  【概要】
    - CodeCommitに新規でリポジトリを作成する
    - CodePipelineがCodeCommitの特定のリポジトリの変更を検知してS3にZipファイルをアップロードするようにする
    - CodeBuildがS3のZipファイルからDockerイメージをECRに保存する
    - CodePipelineがビルドされたDockerイメージを検出してCodeDeployに渡し、ECS(Fargate)にDockerコンテナを起動（デプロイ）するようにする  
    ※ざっくりと記載しているので誤っている箇所があればご指摘ください。
#### 1. CodeCommitからリポジトリを作成する
  1. AWSにアクセスし、サービスから「CodeCommit」を選択する
  2. サイドメニューの[ソース > リポジトリ]から「リポジトリを作成」をクリックする
  3. [リポジトリ名]を「github-mirroring-repo」とする
  4. [説明]には「repository for constructing cicd architecture」と入力する
  5. 作成する

#### 2. Elastic Container Registryにてリポジトリを作成する
  1. AWSにアクセスし、サービスから「ECR」を選択する
  2. サイドメニューの[ソース > リポジトリ]から「リポジトリを作成」をクリックする
  3. [一般設定]セクションより「プライベート」を選択し、[リポジトリ名]を「github-cicd-repo」と入力する
  4. リポジトリを作成する

#### 3. CloudFormationにてVPC等のネットワークを作成する
  1. CloudFormationに使用するymlファイルを作成する
     ```
     ---
     AWSTemplateFormatVersion: 2010-09-09
     Description:  create Simple VPC networks template.
      
     Parameters:
       EnvironmentName:
         Description: An environment name that is prefixed to resource names
         Type: String
         Default: github-cicd
      
       VpcCIDR:
         Description: Please enter the IP range (CIDR notation) for this VPC
         Type: String
         Default: 10.192.0.0/16
      
       PublicSubnetCIDR:
         Description: Please enter the IP range (CIDR notation) for the public subnet in the first Availability Zone
         Type: String
         Default: 10.192.10.0/24
      
       PrivateSubnetCIDR:
         Description: Please enter the IP range (CIDR notation) for the private subnet in the first Availability Zone
         Type: String
         Default: 10.192.20.0/24
      
      
     Resources:
       VPC:
         Type: AWS::EC2::VPC
         Properties:
           CidrBlock: !Ref VpcCIDR
           EnableDnsSupport: true
           EnableDnsHostnames: true
           Tags:
             - Key: Name
               Value: !Sub "${EnvironmentName}-vpc"
      
       InternetGateway:
         Type: AWS::EC2::InternetGateway
         Properties:
           Tags:
             - Key: Name
               Value: !Sub "${EnvironmentName}-igw"
     
       InternetGatewayAttachment:
         Type: AWS::EC2::VPCGatewayAttachment
         Properties:
           InternetGatewayId: !Ref InternetGateway
           VpcId: !Ref VPC
      
       PublicSubnet:
         Type: AWS::EC2::Subnet
         Properties:
           VpcId: !Ref VPC
           AvailabilityZone: !Select [ 0, !GetAZs '' ]
           CidrBlock: !Ref PublicSubnetCIDR
           MapPublicIpOnLaunch: true
           Tags:
             - Key: Name
               Value: !Sub ${EnvironmentName} Public Subnet (AZ1)
      
       PrivateSubnet:
         Type: AWS::EC2::Subnet
         Properties:
           VpcId: !Ref VPC
           AvailabilityZone: !Select [ 0, !GetAZs  '' ]
           CidrBlock: !Ref PrivateSubnetCIDR
           MapPublicIpOnLaunch: false
           Tags:
             - Key: Name
               Value: !Sub ${EnvironmentName} Private Subnet
      
       PublicRouteTable:
         Type: AWS::EC2::RouteTable
         Properties:
           VpcId: !Ref VPC
           Tags:
             - Key: Name
               Value: !Sub ${EnvironmentName} Public Routes
      
       DefaultPublicRoute:
         Type: AWS::EC2::Route
         DependsOn: InternetGatewayAttachment
         Properties:
           RouteTableId: !Ref PublicRouteTable
           DestinationCidrBlock: 0.0.0.0/0
           GatewayId: !Ref InternetGateway
      
       PublicSubnetRouteTableAssociation:
         Type: AWS::EC2::SubnetRouteTableAssociation
         Properties:
           RouteTableId: !Ref PublicRouteTable
           SubnetId: !Ref PublicSubnet
      
       PrivateRouteTable:
         Type: AWS::EC2::RouteTable
         Properties:
           VpcId: !Ref VPC
           Tags:
             - Key: Name
               Value: !Sub ${EnvironmentName} Private Routes
     
       PrivateSubnetRouteTableAssociation:
         Type: AWS::EC2::SubnetRouteTableAssociation
         Properties:
           RouteTableId: !Ref PrivateRouteTable
           SubnetId: !Ref PrivateSubnet
     
       SecurityGroup:
         Type: AWS::EC2::SecurityGroup
         Properties:
           GroupName: "github-cicd-app-sg"
           GroupDescription: "Security group for github-cicd-app"
           SecurityGroupIngress:
             - IpProtocol: tcp
               FromPort: 80
               ToPort: 80
               CidrIp: 0.0.0.0/0
             - IpProtocol: tcp
               FromPort: 443
               ToPort: 443
               CidrIp: 0.0.0.0/0
             - IpProtocol: tcp
               FromPort: 22
               ToPort: 22
               CidrIp: 122.18.246.31/32
           VpcId: !Ref VPC
           Tags:
             - Key: Name
               Value: !Sub "${EnvironmentName}-sg"
      
     Outputs:
       VPC:
         Description: A reference to the created VPC
         Value: !Ref VPC
      
       PublicSubnet:
         Description: A reference to the public subnet in the 1st Availability Zone
         Value: !Ref PublicSubnet
      
       PrivateSubnet:
         Description: A reference to the private subnet in the 1st Availability Zone
         Value: !Ref PrivateSubnet
      
       SecurityGroup:
         Description: Security group for github cicd app
         Value: !Ref SecurityGroup
     ```
  2. AWSにアクセスし、サービスから「CloudFormation」を選択する
  3. サイドメニューの「スタック」から「スタックの作成 > 新しいリソースを使用（標準）」をクリックする
  4. [前提条件-テンプレートの準備]から「テンプレートの準備完了」を選択し、[テンプレートの指定]から「テンプレートファイルのアップロード」を選択する
  5. 「1」で作成したCloudFormationのymlファイルをアップロードし、「次へ」を選択する
  6. [スタック名]は「github-cicd-fargate」とする
  7. 特に変更せず「次へ」を選択する
  8. 「送信」をクリックする
  9. ステータスが「CREATE_COMPLETE」になることを確認する

#### 4. ECSのタスクを実行するポリシーを作成する
  1. AWSにアクセスし、サービスから「IAM」を選択する
  2. サイドメニューの「ロール」から「ロールを作成」をクリックする
  3. [信頼されたエンティティを選択]から「AWSのサービス」を選択しする
  4. [ユースケース]から「Elastic Container Service」の「Elastic Container Service Task」を選択する
  5. [次へ]をクリックする
  6. [許可ポリシー]から「ECS」で検索し、「AmazonECSTaskExecutionRolePolicy」を選択して[次へ]をクリックする
  7. [ロール名]を「github-cicd-ecs-role」、説明はデフォルトのままとする
  8. ロールを作成する

#### 5. ECSでコンテナのオーケストレーションサービスを設定する
  1. AWSにアクセスし、サービスから「ECS」を選択する
  2. サイドメニューから「タスク定義」を選択し、「新しいタスク定義の作成 > JSONを使用した新しいタスク定義の作成」をクリックする
  3. タスク定義を下記のように入力する
     ```
      {
          "requiresCompatibilities": [
              "FARGATE"
          ],
          "family": "github-cicd-app-fargate",
          "containerDefinitions": [
              {
                  "name": "github-cicd-app-task",
                  "image": "AWS_ACCESS_KEY_ID.dkr.ecr.AWS_DEFAULT_LEGION.amazonaws.com/repository_name:tag_name",
                  "portMappings": [
                      {
                          "name": "github-cicd-app-80-tcp",
                          "containerPort": 80,
                          "hostPort": 80,
                          "protocol": "tcp"
                      }
                  ],
                  "essential": true
              }
          ],
          "taskRoleArn": "arn:aws:iam::AWS_ACCESS_KEY_ID:role/DockerCiCdECSRole",
          "executionRoleArn": "arn:aws:iam::AWS_ACCESS_KEY_ID:role/DockerCiCdECSRole",
          "networkMode": "awsvpc",
          "memory": "3 GB",
          "cpu": "1 vCPU",
          "runtimePlatform": {
              "cpuArchitecture": "ARM64"
          }
      }
     ```
  4. [作成]をクリックする
  5. サイドメニューから「クラスター」を選択し「クラスターの作成」をクリックする
  6. [クラスター名]を「github-cicd-cluster」とする
  7. [インフラストラクチャ]は「AWS Fargate」として、作成を行う
  8. クラスター作成後、詳細画面から「サービス」タグを選択し、「作成」をクリックする
  9. [デプロイ設定]セクションの「ファミリー」は「github-cicd-app-fargate」、「サービス名」は「docker-cicd-app-service」を入力する
  10. [必要なタスク]は一旦「0」にしておく
  11. [ネットワーキング]セクションより、「VPC」は「github-cicd-vpc」、「サブネット」は「public」のみ選択し、  
      「セキュリティグループ」は「default」と「作成したSG」を、「パブリックIP」は「on」にしておく
  12. [作成]をクリックする

#### 6. CodeBuildがDockerイメージをビルドする際に使用するymlファイルを作成する
  以下の内容を記載する
  ```
  version: 0.2
  phases:
    pre_build:
      commands:
        - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin [リポジトリURLのリポジトリ名を排除した文字列]
        - REPOSITORY_URI=[リポジトリURI]
        - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
        - IMAGE_TAG=${COMMIT_HASH:=latest}
        - CONTAINER_NAME="php-apache"
    build:
      commands:
        - docker build -t $REPOSITORY_URI:latest .
        - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
    post_build:
      commands:
        - docker push $REPOSITORY_URI:latest
        - docker push $REPOSITORY_URI:$IMAGE_TAG
        - printf '{"name":"%s","imageUri":"%s"}' "$CONTAINER_NAME" "$REPOSITORY_URI:$IMAGE_TAG" > imagedefinitions.json   （※１）
  artifacts:
    files: imagedefinitions.json
  ```
  ※１：imagedefinitions.jsonファイルを出力するコマンドの書き方はAWS公式を参照すること！
  https://docs.aws.amazon.com/ja_jp/codepipeline/latest/userguide/ecs-cd-pipeline.html

  【変数の記載内容】
  - CONTAINER_NAME：ECSのタスク定義のサービスに記載しているJSONのContainerの名称
  - REPOSITORY_URI：Amazon ECRリポジトリURI

  ======重要！======  
  AWSの公式にも記載がある通り、Amazon ECSによるデプロイメントを行う際は、
  イメージを定義するJSONファイルとして「imagedefinitions.json」が必要となる。
  ```
  https://docs.aws.amazon.com/codepipeline/latest/userguide/file-reference.html
  ```
  buildspec.ymlのpost_buildセクションに、以下の記述を追加すること！！！
  - printf '[{"name":"next-container-task","imageUri":"%s"}]' $REPOSITORY_URI:latest > imagedefinitions.json

#### 7. Dockerfileを作成する
  今回はphpとApacheの実行環境が構築できるDockerの公式イメージを使用する
  ```
  FROM php:8.1-apache
  COPY src/ /var/www/html/
  ```

#### 8. コンテナビルド用ファイル一式をローカルからpushする
  ```
  git add -A
  git commit -m "push files for constructing docker images"
  git push
  ```

#### 9. CodePipelineを構築する
  1. AWSにアクセスし、サービスから「CodePipeline」を選択する
  2. サイドメニューから「パイプライン > パイプライン」を選択し、「パイプラインを作成する」をクリックする
  3. [パイプライン名]は「github-cicd-pipeline」とする
  4. [サービスロール]は「新しいサービスロール」を選択し、「次に」をクリックする
  5. [ソースステージを追加する]のセクションにて[ソースプロバイダー]より「github（バージョン2）」を選択する
  6. [GitHubに接続する]をクリックする
  7. [接続名]を「github-repo」とする
  8. [Authorize]すると「新しいアプリをインストールする」ボタンが表示されるのでクリックする
  9. 目的のリポジトリのみを選択してinstallする
     ※このときに誤ったリポジトリ名を指定しないように注意すること！
  11. 接続をクリックする
  12. [リポジトリ名]は「docker-cicd-pipeline」、[ブランチ名]は「master」を選択する
  13. [パイプライントリガー]は「ブランチにプッシュイン」を選択する
  14. [ビルドステージを追加する]のセクションにて[ソースプロバイダー]より「AWS CodeBuild」を選択する
  15. [プロジェクトを作成する]をクリックする
  16. [プロジェクト名]は「github-cicd-project」とする
  17. [環境イメージ]は「マネージド型」、[コンピューティング]は「EC2」、[OS]は「Amazon Linux」を選択する
  18. [ランタイム]は「Standard」、[イメージ]は「aws/codebuild/amazonlinux2-aarch64-standard:3.0」を選択する
  19. [特権付与]にチェックを入れて、[イメージのバージョン]は最新を選択する
  20. [ログ]セクションの[CloudWatch]のチェックは外す
  21. [CodePipelineに進む]をクリックする
  22. [次に]をクリックする
  23. [デプロイステージを追加する]のセクションにて[ソースプロバイダー]より「Amazon ECS」を選択する
  24. [クラスター名]は「github-cicd-cluster」、[サービス名]は「docker-cicd-app-service」を選択する
  25. [次に]をクリックする
  26. [パイプラインを作成する]をクリックする

  ※上記完了後、パイプラインが実行されるが、おそらくcodebuildのセクションでエラーが発生する（権限の問題）

#### 10.CodeBuildの権限を変更する
  1. AWSにアクセスし、サービスから「CodeBuild」を選択する
  2. サイドメニューから「ビルド」を選択し、「github-cicd-project」の詳細画面を開く
  3. [ビルドの詳細]タブを開き、[サービスロール]をクリックする
  4. ECRへのアクセス権限とSSMへの読み取り権限を付与する
     - AmazonEC2ContainerRegistryPowerUser
     - AmazonSSMReadOnlyAccess

#### 11.再度パイプラインを実行する
  1. AWSにアクセスし、サービスから「ECS」を開く
  2. サイドメニューから「クラスター」を選択し、詳細画面より、[サービス]タブの「docker-cicd-app-service」をチェックし、
     [更新]ボタンを押下する
  3. [必要なタスク]を「1」に変更し、[更新]をクリックする
  4. サービスから「CodePipeline」を選択する
  5. 該当のパイプラインを選択し、「変更をリリースする」をクリックする
  6. 各ステージが問題なく「success」になることを確認する

#### 12. おまけ：パイプラインの実行通知をslackに送る
  ■ Teamsに通知する場合の参考サイト
  ```
  https://zuntan02.hateblo.jp/entry/2021/04/19/143523
  ```

  ■ Slackの場合の参考サイト
  ```
  https://dev.classmethod.jp/articles/notify-codepipeline-events-to-slack/
  ```

## パターン2. GitHub ActionsによるCI/CD構築
### 1. 

## まとめ
  今回は、「GitHubとAWSのCode系サービスを利用したCI/CD構築」と「GitHub Actionsを用いたCI/CD構築」の２種類を検証した。
  技術的にはどちらも勉強になったが、管理という面でいえばGitHub Actionsがよさそうだと感じた。  
  
  ただし、求められる要件次第で、どちらの構成にした方が良いかを考える必要があり、
  メリットやデメリットを把握しておくことが重要だと感じた。
  
