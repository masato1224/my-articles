---
title: 'CUDOS構築記 Vol.2: CURの集約先を作成する'
tags:
  - AWS
  - Terraform
  - CFM
  - CUDOS
  - CloudIntelligenceDashboards
private: false
updated_at: '2025-01-10T15:28:43+09:00'
id: 015b4c9f5bdeaeeb5cf2
organization_url_name: ateam-life-design
slide: false
ignorePublish: false
---

こちらの記事は、[CUDOS構築記 Vol.1: CUDOSの概要と構築方法の検討](https://qiita.com/masatomasato1224/items/311e890ade9b48700cbe)の続きです。
前回は、CUDOSの概要と構築方法の検討について紹介しました。今回は、[Step1:CUR集約先の作成](https://qiita.com/masatomasato1224/items/311e890ade9b48700cbe#%E6%A7%8B%E7%AF%89%E3%82%B9%E3%83%86%E3%83%83%E3%83%97:~:text=%E3%81%8C%E3%81%82%E3%82%8A%E3%81%BE%E3%81%99%E3%80%82-,CUR%E3%81%AE%E9%9B%86%E7%B4%84%E5%85%88%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B,-CUR%E3%81%AE%E5%87%BA%E5%8A%9B)についてコードを交えながら解説します。

## 本記事の構成

- 前回のおさらい
- 構築範囲の確認
- 構築手順

## 前回のおさらい

- CUDSの構築ステップは3つ
  - Step1: CURの集約先を作成する
  - Step2: CURの出力と集約先へのレプリケーション設定をする
  - Step3: CURの出力と集約先へのレプリケーション設定をする
- 構築先アカウント
  - Step1,3:ダッシュボードを設置するアカウント
  - Step2: 可視化対象アカウントすべて
- CUDOSの構築方針
  - Step1,2はTerraformで構築
  - Step3はCLI（cid-cmd tool）で構築

## 構築範囲の確認

![構築範囲](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/4c63ffe7-51dc-28ef-6366-4d02eab2454f.png)

赤枠で囲ってある範囲が、本記事で構築する対象です。
CURの集約先を作り、レプリケーションに必要なポリシーを設定します。

## 構築に必要な情報

- ダッシュボードを設置するAWSアカウントID
- すべての可視化対象AWSアカウントID

## 構築作業

### 前提

- Terraformのplan/applyがローカルPCで実行できる
- TerraformのRemote Backendが準備されている
- AWS CLIのインストールと認証が行える
- リポジトリが作成されており、ローカルにCloneされている
- ディレクトリ構成

```bash:ディレクトリ構成
(リポジトリルート)
└── cid_dashboard
```

### 事前準備

作業ディレクトリを用意して、カレントディレクトリを移動します。

```bash
mkdir cur_setup_destination
cd cur_setup_destination
```

### Terraformの初期化

Terraformの設定ファイルを作成します。

```bash
touch backend.tf
touch provider.tf
```

```tf:backend.tf
# 適宜バケット名、キー名、リージョン、プロファイル名を変更してください
terraform {
  backend "s3" {
    bucket  = "tfstate-cost-dashboard"
    key     = "terraform.state.cudos.cur-setup-destination"
    region  = "ap-northeast-1"
    profile = "data-collection-account"
  }
}
```

```tf:provider.tf
# 適宜versionを変更してください
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.73.0"
    }
  }
}

# 適宜profileを変更してください
# ただし、regionに関してはus-east-1を指定してください
# 理由は後述します
provider "aws" {
  region  = "us-east-1"
  profile = "data-collection-account"
}

provider "aws" {
  alias   = "useast1"
  region  = "us-east-1"
  profile = "data-collection-account"
}
```

Terraformの初期化を行います。

```bash
terraform init
```

## 補足: リージョンの指定について

集約先アカウントのCURをCUDOSに連携する場合、リージョンは`us-east-1`を指定してください。[us-east-1以外でCUR関連のリソースの作成](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cur_report_definition)が行えないためです。
集約先アカウントのCURをCUDOSに連携する必要がない場合は、他のリージョンを指定しても問題ありません。ただし、リージョン間のデータ転送料を避けるため、後続のリソースを含めてリージョンは統一することをお勧めします。

## リソースの作成

公式から提供されているTerraformモジュールのラッパーを作成します。

```bash
touch main.tf
touch outputs.tf
```

```tf:main.tf
module "cur_destination" {

  # 適宜バージョンを変更してください
  # see: https://github.com/aws-samples/aws-cudos-framework-deployment/tree/main/terraform-modules/cur-setup-destination
  source = "github.com/aws-samples/aws-cudos-framework-deployment//terraform-modules/cur-setup-destination?ref=0.3.13"

  # 集約アカウント以外で、CUDOSに連携したいアカウントIDを指定してください
  source_account_ids = [
    "000000000000",
    "111111111111",
    "222222222222"
  ]

  # 集約アカウントをCUDOSに連携する場合はtrue、連携しない場合はfalseを指定してください
  create_cur                        = true
  enable_split_cost_allocation_data = true

  providers = {
    aws.useast1 = aws.useast1
  }
}

```

```tf:outputs.tf
# 後続のステップで使用するため、バケットARNを出力します
output "destination_bucket_arn" {
  value = module.cur_destination.cur_bucket_arn
}
```

Terraformのplan/applyを実行します。

```bash
terraform plan
```

```bash
terraform apply
```

AWSコンソールでリソースが作成されていることを確認します。CURに関しては反映まで24時間ほどかかるので、時間をおいて確認してください。

---

以上で、この記事は終了です。
次回の記事では、[Step2:CURの出力と集約先へのレプリケーション設定](https://qiita.com/masatomasato1224/items/311e890ade9b48700cbe#%E6%A7%8B%E7%AF%89%E3%82%B9%E3%83%86%E3%83%83%E3%83%97:~:text=CUR%E3%81%AE%E5%87%BA%E5%8A%9B%E3%81%A8%E9%9B%86%E7%B4%84%E5%85%88%E3%81%B8%E3%81%AE%E3%83%AC%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E8%A8%AD%E5%AE%9A%E3%82%92%E3%81%99%E3%82%8B)について解説します。
※ 現在、執筆中です。

それでは、次回の記事でお会いしましょう。
