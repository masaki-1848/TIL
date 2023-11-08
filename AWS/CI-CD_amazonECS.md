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
     ※cloudFormationを実行するためのymlファイルには、主に①AWSTemplateFormatVersion、②Description、③Mappings、④Parameters、⑤Resourcesの５項目について記載する  
     　記載内容の詳細については、別記事で記載する
     　ちなみに、Windowsのメモ帳アプリでyamlファイルを作成すると改行コードが「CR（キャリッジリターン）+LF（ラインフィード）」となるため、
     　CloudFormationに取り込むとエラーが発生する（Linux形式（LF）にする必要がある！！）
  4. 
  
  
  
