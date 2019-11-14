# handson-cloudformation-create-vpc

# 概要

AWS Cloudformationを用いてVPCをデプロイするハンズオン用リポジトリ

# 目的

- Cloudformationによるインフラストラクチャのデプロイを体験する
- Cloudformationの変更セットを用いてリソースの変更がどのように行われるのか確認する

# 前提条件

- AWS CLIコマンドを用いてデプロイします。AWSCLIが使える環境があることが必要です。
    - ラップトップへのAWSCLIのインストール方法がわからない場合AmazonLinux2を用いてCLIコマンドを発行することをおすすめします。
    
- Cloudfromationを実行するための十分なIAM権限をIAM Roleに付与し、EC2にアタッチしていること。（AmazonLinux２利用時）
    - ラップトップからCLIコマンドを発行する場合は十分な権限を持ったIAMユーザーを作成し、CLIのクレデンシャル情報がセット完了していること
    
- VPCを一つ作成します。DefaultのVPC上限数（Default値：５）に引っかからない数のVPCがあるリージョンで実行すること

- CloudformationのYAMLファイルを格納するため、S3バケットが１つ必要です。事前に作成が完了していること。
    - 同一アカウント、同一リージョンにあるものとします。

# 想定環境

本ハンズオンは特別な事情が無い限り東京リージョンで実行及び検証を行っています。

-----

# create-vpc-template.yml

デプロイされるCloudformationのファイル。
YAML記法で記載されており、VPCの構成情報が定義されている。
今回はこのファイルをデプロイし、VPCを作成する。

# update-vpc-template.yml

環境を更新するために2回目に実行するテンプレートファイル
セキュリティグループやIAMロールの追加・修正を行う

---

# Deploy手順

ここからの手順はターミナルを使用します。
ターミナルのカレントディレクトリは当マークダウンテキスト(README.md)があるのと同一の階層でコマンドを実行してください。
十分な権限を持った状態でCLIコマンドを発行できることを前提としています。
本ハンズオン内に限り、IAM権限固有の問題で発生するエラーを回避する目的で `Administrator` に準ずる権限を利用することをお勧めます。

**実際の環境では最小権限の原則に即して不用意に強い権限を与える事はしないことを強く勧めます。**

1. 環境変数の設定

デプロイに必要な変数を環境変数に設定します。

```bash
# 使用するリージョン
export AWS_DEFAULT_REGION=ap-northeast-1
# Cloudformationに渡すパラメータの値
S3_BUCKET_NAME=<作成したS3バケットのバケット名>
MAIN_STACK_NAME=CFn-Demo-$(date "+%Y%m%d-%H%M%S")
CIDER=10.100.0.0/16
az1=ap-northeast-1a
az2=ap-northeast-1d
PublicSubnetCIDER01=10.100.1.0/24
PublicSubnetCIDER02=10.100.2.0/24
```

上記変数を修正後、ターミナルに貼り付けます。
CloudformationテンプレートをS3バケットにアップロードします。
今回は単一のテンプレートですが、環境が大きくなった際に複数のテンプレートの依存関係をまとめてアップロード可能な `package` コマンドを使用します。

```bash
aws cloudformation package \
    --template-file ./create-vpc-template.yml \
    --s3-bucket ${S3_BUCKET_NAME} \
    --s3-prefix ${MAIN_STACK_NAME} \
    --output-template-file ./deploy-create-vpc-template.yml
```

マネジメントコンソールにログインし、S3バケットを確認してください。
テンプレートファイルが追加されているはずです。

```bash
aws cloudformation deploy \
    --template-file ./deploy-create-vpc-template.yml \
    --s3-bucket ${S3_BUCKET_NAME} \
    --s3-prefix ${MAIN_STACK_NAME} \
    --stack-name ${MAIN_STACK_NAME} \
    --parameter-overrides CIDER=${CIDER} az1=${az1} az2=${az2} PublicSubnetCIDER01=${PublicSubnetCIDER01} PublicSubnetCIDER02=${PublicSubnetCIDER02}
```

Cloudfromationのデプロイコマンドを利用してスタックを作成します。
コマンドをターミナルに入力後、マネジメントコンソールでCloudfomationのスタックを確認してください。
しばらく時間がかかかりますが、緑色で作成が完了したことが確認できれば成功です。
実際にVPCを確認して環境のデプロイが完了していることを確認してください。

ここまでで、Cloudformationを用いたVPC環境の構築自動化が完了しました。
複数のアカウントを管理している際などに、事前にCloudformationを用いてVPCを作成してからアカウントを払い出す。などを行うことにより構築作業の負荷を下げることや、設定のミスを防ぐ効果が期待できます。

---

# 変更セットの確認

Cloudfornationを用いて環境を更新する際は現存の環境に削除を伴う変更が発生するかを考えて実行することが大切です。
例えば、セキュリティグループの名前は一度作成したら変更することはできません（気になる人はマネジメントコンソールで確認してみましょう）
しかし、Cloudformationではセキュリティグループの名前を変更し、再度デプロイをすることで名前の変わったセキュリティグループを確認できます。
これは実際には一度古いセキュリティグループを削除し新しいセキュリティグループを作成して置換する。という挙動を示しています。

意図しない削除を伴う置換は障害を引き起こす原因になりえます。
ここでは変更セットを利用して、デプロイの際にどのような種類の変更が起こるのかを確認してから更新を行う方法を見ていきたいと思います。

`update-vpc-template.yml` という先ほどのテンプレートを更新したYAMLファイルを用いて更新を行います。

このファイルでは、先にデプロイした環境に以下の変更を加えています。

- セキュリティグループの名前を変更する
- セキュリティグループのインバウンドのルールにHTTPSのany設定を追加する
- IAM Roleを１つ追加で作成する

更新したテンプレートをS3に格納する

```bash
aws cloudformation package \
    --template-file ./update-vpc-template.yml \
    --s3-bucket ${S3_BUCKET_NAME} \
    --s3-prefix ${MAIN_STACK_NAME}-update \
    --output-template-file ./deploy-update-vpc-template.yml
```

更新したファイルが格納されているはずです。
S3のバケットを確認してください。先程と差異があるはずです。

次に、変更セット(change set)を作成します。

```bash
CHANGE_SET=CFn-Demo-$(date "+%Y%m%d-%H%M%S")-change-set
aws cloudformation create-change-set \
    --stack-name ${MAIN_STACK_NAME} \
    --change-set-name=${CHANGE_SET} \
    --template-body file://deploy-update-vpc-template.yml \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
    --parameters ParameterKey=CIDER,ParameterValue=${CIDER},UsePreviousValue=false ParameterKey=az1,ParameterValue=${az1},UsePreviousValue=false ParameterKey=az2,ParameterValue=${az2},UsePreviousValue=false ParameterKey=PublicSubnetCIDER01,ParameterValue=${PublicSubnetCIDER01},UsePreviousValue=false ParameterKey=PublicSubnetCIDER02,ParameterValue=${PublicSubnetCIDER02},UsePreviousValue=false
```

上記のコマンドをターミナルで実行すると変更セットが作成されます。
Cloudformationで変更セットを確認してください。
modifyやaddといった情報が確認できるはずです。また **** でtrueが確認できると思います。
これがtrueの場合は削除を伴う置換が発生することを示しているので注意が必要です。

では、確認ができ変更が想定どおりだったのでデプロイを行います。
change setの実行をすることで作成したスタックに変更が適応されます

```bash
aws cloudformation execute-change-set \
    --stack-name ${MAIN_STACK_NAME} \
    --change-set-name ${CHANGE_SET}
```

マネジメントコンソールでセキュリティグループを確認すると名前が変わっており、HTTPSの設定が増えていることが確認できると思います。
また、EC2用のロールも新規で作成されていることが確認できたと思います。

以上でCloudformationを運用する上で変更を行う際にどのような変更が起こるかを確認するChange setの使い方の確認ができました。

--- 

環境の削除

マネジメントコンソールのCloudformationのスタックから今回作成したスタックの削除を実行してください。
環境が削除されます。

以上でハンズオンは終了です。
お疲れさまでした。
