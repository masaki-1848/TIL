# AWS CI/CD for Amazon ECS ハンズオンを試してみる
  ## 参考サイト
  [https://qiita.com/mksamba/items/ffbdbd0a8ca25c3e060e](https://pages.awscloud.com/rs/112-TZM-766/images/AWS_CICD_ECS_Handson.pdf)

  ## 目的
  Code系のサービス（CodeCommit、CodeDeploy、CodePipeline）を用いて、EC2インスタンスにローカルリポジトリからの  
  pushをトリガーとして自動デプロイする仕組みを構築することができたので、次は、Dockerイメージを自動的に登録し  
  Dockerイメージからコンテナを自動的にデプロイする仕組みを作りたいと思い技術を検証するために実施

  ## 今回挑戦するアーキテクチャ


  ## 操作手順
  > ### 1.ECS環境の構築
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
  1. ECRにログインする
  > 【ECR（Elastic Container Registry）のリポジトリを作成する】の手順５でメモしたコマンドを使用すること！
  - ログイン
  ```
  aws ecr get-login-password --region ～～
  ```
  
  - タグ付け
  ```
  docker tag repositry_name:latest ○○.dkr.ecr.region_name.amazonaws.com/repository_name:latest
  ```

  - イメージのpush
  ```
  docker push ○○.dkr.ecr.region_name.amazonaws.com/repositry_name:latest
  ```

  
