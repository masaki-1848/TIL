![image](https://github.com/masaki-1848/TIL/assets/63137407/e06159dd-95df-45d7-8823-87eb7720d872)# CodeCommitを用いたEC2のデプロイメント構築


 ## 【参考サイト】
       https://docs.aws.amazon.com/ja_jp/codepipeline/latest/userguide/tutorials-simple-codecommit.html
 ## 【手順】
  ### 1.CodeCommitにアクセスし、create Repositoryから新規リポジトリを作成する
  リージョンは作業用リージョンに切り替えること
  ### 2.Gitの認証情報を作成する
  IAMのユーザ管理サービスから、git cloneする際に認証ユーザーを検索し、管理画面から「セキュリティ認証情報」タブを選択して、
  下部にある「AWS CodeCommit の HTTPS Git 認証情報 」セクションより、認証情報を生成する
  ⇒　生成した認証情報をダウンロードして、git cloneコマンドを実行する際の認証情報を入力する
  ### 3.作成したリポジトリをローカルにgit cloneする
  コマンド例
  ```
  git clone https://git-codecommit.us-east-1.amazonaws.com/SampleAppRepo C:\path\to\work\directory
  ```
  コマンド実行時には、「2.」の手順でダウンロードした認証情報を使用する 
  ### 4.Laravelの新規プロジェクトをルート直下に作成する
  ```
  composer create-project laravel/laravel sample-app
  ```
  ### 5.gitにすべてのファイルをアップロードする
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
  
  ### 6. デプロイ先のEC2インスタンスを作成する
  #### 【事前準備】
  EC2インスタンスを起動する先のVPCとサブネット、セキュリティグループを作成しておくこと
  ##### １. VPC作成
  1. サービスからVPCを選択する
  2. VPCを作成するを選択する
  3. [名称]に「sample-app-vpc-01」、IPv4のCIDRブロックを「10.0.0.0/16」として作成する
  ##### ２. サブネット作成
  1. VPCのサイドメニューから[サブネット]を選択する
  2. [サブネットを作成]を選択する
  3. VPCから先程作成したVPC（例：sample-app-vpc-01）を選択する
  4. [サブネット名]に「sample-app-pub-subnet-01」を入力し、[アベイラビリティーゾーン]から一番上の選択肢を選択する
  5. [IPv4 subnet CIDRブロック]に「10.0.1.0/24」を入力する
  6. サブネットを作成する
  ##### ３. セキュリティグループを作成する
  1. VPCのサイドメニューから[セキュリティグループ]を選択する
  2. [セキュリティグループを作成]を選択する
  3. [セキュリティグループ名]を「sample-app-sg-01」、[説明]に「security group for sample app」、[VPC]から先程作成したVPCを選択する
  4. [インバウンドルール]セクションより、下記２項目を追加する
    - SSH / TCP / port:22 / マイIP
    - HTTP / TCP / port:80 / マイIP
  ##### ４. ルートテーブルを作成する
  1. VPCのサイドメニューから[ルートテーブル]を選択する
  2. [ルートテーブルを作成]を選択する
  3. [名前]に「sample-app-route-table-01」を入力し、[VPC]から「先程作成したVPC」を選択する
  4. ルートテーブルを作成後、[アクション]から「サブネットの関連付けを編集」を選択する
  5. 先程作成したサブネット「sample-app-pub-subnet-01」を選択して、関連付けを保存する
  6. [アクション] から「ルートを編集」を選択する
  7. [追加] を選択し、「0.0.0.0/0」で「インターネットゲートウェイ」、５で作成するインターネットゲートウェイ「sample-app-igw-01」を選択して登録する
  ##### ５. インターネットゲートウェイを作成する
  1. VPCのサイドメニューから[インターネットゲートウェイ]を選択する
  2. [インターネットゲートウェイの作成]を選択する
  3. [名前]に「sample-app-igw-01」を入力し、作成する
  4. [アクション]から「VPCにアタッチ」を選択する
  5. 先程作成したVPCを選択して、インターネットゲートウェイのアタッチを行う
  #### 【EC2のロール作成】
  ※スポットインスタンスを使用する場合は既に、「AWSServiceRoleForEC2Spot」が存在するのでそちらを使用すること
  1. https://console.aws.amazon.com/iam/ で IAM コンソール を開く
  2. コンソールダッシュボードで [ロール] を選択する
  3. [ロールの作成] を選択する
  4. [信頼できるエンティティのタイプを選択] より「AWSのサービス」を選択する
  5. [ユースケース] より「EC2」を選択し、[次へ] を選択する 
  6. 【AmazonEC2RoleforAWSCodeDeploy】と【AmazonEC2RoleforAWSCodeDeploy】ポリシーを選択し、[次へ] を選択する
  7. [ロール名] を「sample-app-ec2-role」を入力し、作成する
  #### 【インスタンス作成】
  1. https://console.aws.amazon.com/ec2/でAmazon EC2 コンソールを開く
  2. [インスタンスを起動] を選択する
  3. [名前] に「sample-app-pipeline-01」を入力する
  4. [アプリケーションと OS イメージ (Amazon マシンイメージ)]からAmazon Linux 2 AMIを選択する
  5. [インスタンスタイプ] から「t2.micro」インスタンスのハードウェア構成として無料利用枠の対象タイプを選択する
  6. ネットワークの割り当てセクション
   - 「パブリックIPの自動割り当て」で、ステータスを「有効」にする
   -  [セキュリティグループの割り当て] の横にある [新規セキュリティグループを作成]
   -  [SSH] の行の [ソースタイプ] で、[My IP] を選択する
   -  [セキュリティグループの追加] を選択し、[HTTP] を選択し、[ソースタイプ] で [My IP] を選択する
  7. [Advanced Details] (高度な詳細) を展開し、IAM インスタンスプロファイルで、前の手順で作成した IAM ロール（sample-app-ec2-role）を選択する
  8. [インスタンス数] は「１」を入力し、インスタンスを起動する
  #### 【EC2インスタンスにCodeDeployのサービスをインストールする】
  ※自動化したい場合は、AWS Systems Mangerにて実行する方法もある
  1. EC2インスタンスにTera-termなどのSSHクライアントからアクセスする
     ログインuserは「ec2-user」
  2. ログイン後に下記のコマンドを実行する
     ```
     $ sudo yum install ruby
     $ wget https://aws-codedeploy-ap-northeast-1.s3.ap-northeast-1.amazonaws.com/latest/install
     $ chmod +x ./install
     $ sudo ./install auto
     $ sudo service codedeploy-agent status
     ```
  3. 
  #### 【CodeDeployのロール作成】
  1. CodeDeployインスタンスへのエージェントのインストールと管理を可能にするインスタンスロールを作成する
     https://console.aws.amazon.com/iam/ で IAM コンソール を開く
  2. コンソールダッシュボードで [ロール] を選択する
  3. [ロールの作成] を選択する
  4. [信頼できるエンティティのタイプを選択] より「AWSのサービス」を選択する
  5. [ユースケース] より「CodeDeploy」を選択し、[次へ] を選択する
  6. 【AWSCodeDeployRole】ポリシーが選択されていることを確認し、[次へ] を選択する
  7. [ロール名] を「sampele-app-codedeploy-role」と入力し、作成する
  #### 【CodeDeployのアプリケーション作成】
  1. https://console.aws.amazon.com/codedeploy CodeDeploy でコンソールを開く
  2. [CodeDeploy] のサイドメニューから「デプロイメント」を選択し「アプリケーションの作成」を実行する
  3. [アプリケーション名] に、「sample-app」と入力する
  4. [コンピューティングプラットフォーム] で [EC2/オンプレミス] を選択する
  5. アプリケーションを作成する
  6. [デプロイグループの作成] を選択する
  7. [デプロイグループ名] に「sample-app-deploy-group」を入力する
  8. [サービスロール] は「sample-app-codedeploy-role」を選択する
  9. [デプロイタイプ] は、[インプレース] を選択する
  10. [環境設定] で、[Amazon EC2 インスタンス] を選択する
  11. [キー] フィールドに [Name] を入力する
  12. [値]  フィールドに「sample-app-pipeline-01」を入力する
  13. [AWS Systems Manager を使用したエージェント設定] で「今すぐ実行」を選択し、アップデートを行う
  14. [デプロイ設定] で、[CodeDeployDefault.OneAtATime] を選択する
  15. [ロードバランサー] で、[ロードバランシングの有効化] が選択されていないことを確認する
  16. デプロイグループを作成する
  ### 7. CodePipelineで最初のアプリケーションを作成する
  1. https://console.aws.amazon.com/codepipeline/ CodePipeline でコンソールを開く
  2. [パイプラインの作成] を選択する
  3. [パイプライン名] に「sample-app-pipeline」と入力する
  4. [パイプラインタイプ] は「V2」を選択する
  5. [サービスロール] は「新しいサービスロール」を選択し、[次へ] を選択する
  6. [ソースプロバイダ] で、[CodeCommit] を選択する
  7. [リポジトリ名] は、作成した CodeCommit リポジトリの名前を選択する
  8. [ブランチ名] は、[master] を選択し、[次へ] を選択する
  9. [ビルドステージを追加する] セクションは「スキップ」を選択する
  10. [デプロイプロバイダ] は、[CodeDeploy] を選択する
  11. [アプリケーション名] は、先ほどCodeDeployで作成した「sample-app」を選択する
  12. [デプロイグループ] は、先ほどCodeDeployで作成した「sample-app-deploy-group」を選択する
  13. パイプラインを作成する
  ### 8. pipelineの実行状況を確認する
  1. CodePipelineのサービスより、先ほど作成した「sample-app-pipeline」のステータスを確認する
     - エラーになっている場合
       - EC2インスタンスが正しく起動または実行されているか？
       - CodeDeployのヘルスチェックが正しく構成されているか？
       - CodeDeployのデプロイグループが適切に構成されているか？
       - デプロイプロセスのログを確認して、エラーの原因はどうなっているか？
  #### 調査方法
  1. EC2インスタンスにリモートからSSHでアクセスする
     - codedeploy-agent プロセスが起動しているか確認する
       ```
       ps -ef | grep codedeploy-agent | grep -v grep
       service codedeploy-agent status
       ```
       サービスが起動していない場合は、下記のコマンドを実行する
       ```
       service codedeploy-agent start
       ```
       「The AWS CodeDeploy agent is running as PID XXXXX」と表示されればエージェントが正常に動いている。
  2. ログを確認する
     ```
     view /var/log/aws/codedeploy-agent/codedeploy-agent.log
     ```
     ログを確認しても原因が分からない場合は、codedeploy-agentサービスの再起動およびEC2インスタンスを再起動を試してみる。
     ```
     sudo service codedeploy-agent restart
     ```
     下記のエラーメッセージが表示される場合は、EC2インスタンスに対してS3へのフルアクセスIAMロールを付与して様子を見る
     ```
     CodeDeploy agent was not able to receive the lifecycle event. Check the CodeDeploy agent logs on your host and make sure the agent is running and can connect to the CodeDeploy server.
     ```
  3. 
   ### 9. 
  
