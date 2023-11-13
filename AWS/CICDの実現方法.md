# AWSのサービスを用いたCI/CD実現の最小構成について調べてみた

## 代表的なAWSサービスを用いたCI/CDの流れ

  1. CodeCommit上のリポジトリにローカルリポジトリの変更をpushする
  2. CodePipelineがCodeCommitのリポジトリ変更を検知してS3にリポジトリのzipファイルを出力
  3. CodeBuildがS3のZipファイルからコードのユニットテスト、ビルド等を行う
  4. ビルド完了後、CodePipelineがS3にビルドされたリソースの情報を記載したZipファイルを出力し、
     そのファイルからCodeDeployで本番用サーバーへのデプロイを開始する
  5. CodeDeployが本番用サーバーにビルドされたコードをデプロイする

  ※上記は、ローカルで作業した内容を本番サーバーに自動的に反映したり、プログラムのテストを行ってくれる一連の自動化を実現するサービス構成である

## コンテナオーケストレーションについて

  コンテナ化されたアプリケーションを立ち上げるためにEC2インスタンスやFargate上にデプロイしたり、  
  立ち上がっているコンテナの設定を変更・削除したり、立ち上げるコンテナの数・配置を自動で配分したりなど  
  コンテナのデプロイ・管理・スケールの自動化を行う機能

  AWSでは、ECS(Elastic Container Service)がこれに当たる。

  ■ コンテナアプリケーション公開の流れ
  1. ローカルで作成したコードをコンテナの基になるイメージにして、ECR（Elastic Container Registory）に保存する
  2. タスク定義ファイルが保存されたECRのイメージを参照する
  3. サービスがタスク定義ファイルを参照する
     - サービスは複数定義することができ、クラスターというグループで管理する
  4. サービスがFargate上にタスクを立ち上げる
     - Fargateはコンテナをサーバーレスで実行することができるようにする機能
     - サーバ運用で必要となるOSのメンテナンス等の管理が不要になる
     - ECSがコンテナを起動するタイプとして「EC2」か「Fargate」が選択できる
     - EC2にするとホストマシンの運用と管理が必要になる
  5. インターネットから立ち上げたタスクつまりコンテナのアプリケーションが閲覧される

## 実際のCI/CD構築
  ■ 概要
  1. ローカルPCで作成したアプリケーションをCodeCommit上のリポジトリにpush
  2. CodePipelineがCodeCommitのリポジトリ変更を検知してS3にリポジトリのzipファイルを出力する
  3. CodeBuildがS3のzip化されたリポジトリ内のbuilspec.ymlファイルを基にアプリケーションのイメージ化を行う
  4. CodeBuildが作成されたアプリケーションのイメージをECRに保存し、そのイメージのタグ付きURLをjsonファイルに記載してzip化し、S3に出力する
  5. タスク定義ファイルがECRの指定したURIとjsonファイルのタグ付きURLを基にコンテナ化するイメージを参照する
  6. サービスがイメージとタスク定義ファイルを参照する
  7. サービスがタスクをFargate上に複数立ち上げる
  8. インターネットから立ち上げたタスクつまりコンテナアプリケーション化されたアプリを閲覧できる
     
  ■ 作業手順
  1. Fargateがコンテナを立ち上げるために必要なVPCその他のネットワークリソースを作成する
    - CloudFormationで一括構築する
  2. サービスから「CloudFormation」を選択し、スタックの作成を行う
  3. テンプレートファイルのアップロードを選択し、ローカルで作成した「cloudFormation.yml」ファイルを選択し次へをクリックする
  4. [スタック名]に「docker-cicd-formation」と入力し次へを選択する
  5. 次へを選択し、送信をクリックする
  6. CloudFormationが実行され、CREATE_COMPLETEになることを確認する
  7. サービスから「ECR」を選択し、イメージを保存するリポジトリを作成する
     - 今回は「gitlab-pipeline-repo」を使用する
  8. ECS他コンテナ公開用リソースを作成する
  9. サービスから「IAM」を選択し、[ロール]からロールを作成する
  10. [信頼されたエンティティタイプ]より、AWSのサービスを選択し、[Elastic Container Service]の「Elastic Container Service Task」を選択する
  11. [task]で検索し、「AmazonECSTaskExecutionRolePolicy」を追加して次へをクリックする
  12. [ロール名]を「DockerCiCdECSRole」と入力し、ロールを作成する
  13. サービスから「ECS」を選択する
  14. サイドメニューのタスク定義を選択し、[新しいタスク定義の作成]から「JSONを使用した新しいタスク定義の作成」をクリックする
  15. 以下のように入力する
  ```
  {
      "requiresCompatibilities": [
          "FARGATE"
      ],
      "family": "docker-cicd-app-fargate",
      "containerDefinitions": [
          {
              "name": "docker-cicd-app-task",
              "image": "AWS_ACCESS_KEY_ID.dkr.ecr.AWS_DEFAULT_LEGION.amazonaws.com/repository_name:tag_name",
              "portMappings": [
                  {
                      "name": "docker-cicd-app-80-tcp",
                      "containerPort": 80,
                      "hostPort": 80,
                      "protocol": "tcp"
                  }
              ],
              "essential": true
          }
      ],
      "taskRoleArn": "arn:aws:iam::989959965747:role/DockerCiCdECSRole",
      "executionRoleArn": "arn:aws:iam::989959965747:role/DockerCiCdECSRole",
      "networkMode": "awsvpc",
      "memory": "3 GB",
      "cpu": "1 vCPU",
      "runtimePlatform": {
          "cpuArchitecture": "ARM64"
      }
  }
  ```

  ■ ポイント
  - image：ECRに作成したリポジトリのURIを指定する
  - taskRoleArn：IAMで作成したroleのARNを指定する（詳細ページからコピー可能）
  - executionRoleArn：上記と同様

  16. サイドメニューの「クラスター」を選択し、「クラスターの作成」をクリックする
  17. [クラスター名]を「DockerCiCdAppCluster」とする
  18. [インフラストラクチャ]は「AWS Fargate」として、作成を行う
  19. クラスター作成後、詳細画面からサービスの作成を行う
  20. [コンピューティングオプション]は「キャパシティプロバイダー戦略」、[キャパシティープロバイダー]は「FARGATE」、  
      [プロバイダーのバージョン]を「LATEST」とする
  22. [デプロイ設定]セクションより、[ファミリー]は「docker-cicd-app-fargate」、[サービス名]は「docker-cicd-app-service」、[必要なタスク]は「0」に変更する
  23. [ネットワーキング]セクションより、[VPC]はcloudformationで作成したVPC、[サブネット]は「public」を選択、セキュリティグループはdefaultと作成したSGを選択、パブリックIPはonにする
  24. ローカルでサンプルアプリケーションを作成する
  25. [buildspec.yml]ファイルを下記のように作成する
    ```
    version: 0.2
    
    phases:
      pre_build:
        commands:
          - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin [リポジトリURLのリポジトリ名を排除した文字列]
          - REPOSITORY_URI=[リポジトリURI]
          - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
          - IMAGE_TAG=${COMMIT_HASH:=latest}
      build:
        commands:
          - docker build -t $REPOSITORY_URI:latest .
          - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
      post_build:
        commands:
          - docker push $REPOSITORY_URI:latest
          - docker push $REPOSITORY_URI:$IMAGE_TAG
          - printf '[{"name":"next-container-task","imageUri":"%s"}]' $REPOSITORY_URI:latest > imagedefinitions.json
    artifacts:
      files: imagedefinitions.json
    ```

    ■ ポイント
    - リポジトリのURIやURLは、ECRの詳細ページから確認できる（リポジトリを選択し、[プッシュコマンドの表示]をクリックすると確認可能）
    - 上記はいずれも秘匿化するべき情報のため、取扱いには十分に注意すること
    
  26. Dockerfileを作成する
  27. ローカルからpushする
  28. サービスから[CodePipeline]を選択する
  29. [パイプラインを作成する]をクリックする
  30. [パイプライン名]は「docker-cicd-pipeline」とする
  31. [サービスロール]は「新しいサービスロール」を選択し、デフォルトのまま「次に」をクリックする
  32. [ソースプロバイダー]は「AWS CodeCommit」を選択する
  33. [リポジトリ名]は「gitlab-mirror-repo」、[ブランチ名]は「master」を選択する
  34. [検出オプション]は「Amazon CloudWatch Events (推奨)」を選択し、「次に」をクリックする
  35. [プロバイダーを構築する]は「AWS CodeBuild」を選択する
  36. [プロジェクト名]は「プロジェクトを作成する」をクリックする
  37. [プロジェクト名]を「docker-cicd-app-project」とする
  38. [環境イメージ]は「マネージド型イメージ」、[コンピューティング]は「EC2」、[オペレーティングシステム]は「Amazon Linux」、
      [ランタイム]は「Standard」、[イメージ]は「aws/codebuild/amazonlinux2-aarch64-standard:3.0」、[イメージのバージョン]は最新、
      [特権付与]にチェックを入れて、[サービスロール]は「新しいサービスロール」を選択する
  39. [ビルド仕様]は「buildspecファイルを使用する」を選択する
  40. [ログ]セクションの「CloudWatch Logs - オプショナル」のチェックを外す
  41. [次に]をクリックする
  42. [デプロイプロバイダー]から「Amazon ECS」を選択する
  43. [クラスター名]から「DockerCiCdAppCluster」を選択する
  44. [サービス名]から「docker-cicd-app-service」を選択する
  45. [次に]をクリックする
  46. [パイプラインを作成する]をクリックする
  47. CodeBuildに付与されているロールに対して、ECRへのアクセス権限とSSMへの読み取り権限を付与する
      - AmazonEC2ContainerRegistryPowerUser
      - AmazonSSMReadOnlyAccess
  49. サービスから[CodeBuild]を開き、詳細ページからロールを確認する
  50. [許可を追加]から「ポリシーをアタッチ」を選択し、「AmazonEC2ContainerRegistryPowerUser」「AmazonSSMReadOnlyAccess」ポリシーを追加する
  51. サービスから[ECS]を開き、[クラスター]からサービスを開き、サービスを更新ボタンを押下する
  52. [必要なタスク]を「1」に修正し、更新ボタンを押下する
  53. 
      
