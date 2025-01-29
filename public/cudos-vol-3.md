---
title: 'CUDOS構築記 Vol.3: CURの出力と集約先へのレプリケーション設定をする'
tags:
  - AWS
  - Terraform
  - CFM
  - CUDOS
  - CloudIntelligenceDashboards
private: false
updated_at: '2025-01-29T18:53:36+09:00'
id: 998e89859aeed4db8c66
organization_url_name: ateam-life-design
slide: false
ignorePublish: false
---

こちらの記事は、[CUDOS構築記 Vol.2: CURの集約先を作成する](https://qiita.com/masatomasato1224/items/015b4c9f5bdeaeeb5cf2)の続きです。
前回は、CURの集約先の作成について紹介しました。今回は、[Step2:CURの出力と集約先へのレプリケーション設定](https://qiita.com/masatomasato1224/items/311e890ade9b48700cbe#%E6%A7%8B%E7%AF%89%E3%82%B9%E3%83%86%E3%83%83%E3%83%97:~:text=CUR%E3%81%AE%E5%87%BA%E5%8A%9B%E3%81%A8%E9%9B%86%E7%B4%84%E5%85%88%E3%81%B8%E3%81%AE%E3%83%AC%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E8%A8%AD%E5%AE%9A%E3%82%92%E3%81%99%E3%82%8B)についてコードを交えながら解説します。

## 本記事の構成

- 前提のおさらい
- 構築範囲の確認
- 構築手順

## 前提のおさらい

- CUDSの構築ステップは3つ
  - （作成済み）Step1: CURの集約先を作成する
  - Step2: CURの出力と集約先へのレプリケーション設定をする
  - Step3: データソース、およびダッシュボードを作成する
- 構築先アカウント
  - Step1,3:ダッシュボードを設置するアカウント
  - Step2: 可視化対象アカウントすべて
- CUDOSの構築方針
  - Step1,2はTerraformで構築
  - Step3はCLI（cid-cmd tool）で構築

## 構築範囲の確認

![構築範囲](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/a5b4a13e-7174-8170-55c6-328c428ac2e3.png)

赤枠で囲ってある範囲が、本記事で構築する対象です。
各アカウントに対して、CURの出力設定とレプリケーションの設定をします。

## 構築作業

### 前提

- Step1の構築が完了していること
- 設定先のAWSアカウントに対する適切なアクセス権限を保持していること
- ディレクトリ構成

```bash:ディレクトリ構成
(リポジトリルート)
└── cid_dashboard
    └── cur_setup_destination
```

### 事前準備

作業ディレクトリを用意して、カレントディレクトリを移動します。

```bash
cd cid_dashboard
mkdir cur_setup_source
cd cur_setup_source
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
    key     = "terraform.state.cudos.cur-setup-source"
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

# CURを収集するアカウントをすべてを指定してください
provider "aws" {
  alias   = "account-1"
  region  = "us-east-1"
  profile = "management-account-1"
}
provider "aws" {
  alias   = "account-2"
  region  = "us-east-1"
  profile = "management-account-2"
}
...
provider "aws" {
  alias   = "account-x"
  region  = "us-east-1"
  profile = "management-account-x"
}
```

Terraformの初期化を行います。

```bash
terraform init
```

## remote_stateの設定

CURのレプリケート先のバケットARNを取得するため、remote_stateを設定します。

```bash
touch remote_state.tf
```

```tf:remote_state.tf
data "terraform_remote_state" "cur_setup_destination" {
  backend = "s3"

  config = {
    bucket  = "tfstate-cost-dashboard"
    key     = "terraform.state.cudos.cur-setup-destination"
    region  = "ap-northeast-1"
    profile = "data-collection-account"
  }
}
```

## リソースの作成

公式から提供されているTerraformモジュールのラッパーを作成します。

```bash
mkdir ./modules
touch ./modules/main.tf
touch main.tf
touch outputs.tf
```

```tf:modules/main.tf
variable "destination_bucket_arn" {
}

module "cur_source" {
  source                            = "github.com/aws-samples/aws-cudos-framework-deployment//terraform-modules/cur-setup-source?ref=0.3.13"
  destination_bucket_arn            = var.destination_bucket_arn
  enable_split_cost_allocation_data = true

  providers = {
    aws         = aws
    aws.useast1 = aws
  }
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.73.0"
    }
  }
}
```

```tf:main.tf
# CURを収集するアカウント分のmoduleを定義してください
module "account-1" {
  source                 = "./modules"
  destination_bucket_arn = data.terraform_remote_state.cur_setup_destination.outputs.destination_bucket_arn
  providers = {
    aws = aws.account-1
  }
}

module "account-2" {
  source                 = "./modules"
  destination_bucket_arn = data.terraform_remote_state.cur_setup_destination.outputs.destination_bucket_arn
  providers = {
    aws = aws.account-2
  }
}
...
module "account-x" {
  source                 = "./modules"
  destination_bucket_arn = data.terraform_remote_state.cur_setup_destination.outputs.destination_bucket_arn
  providers = {
    aws = aws.account-x
  }
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

## おまけ

今後の運用を考慮して、アカウントの追加対応が簡単にできるようShellスクリプトを作成しました。
`provider`と`module`の設定を追加できます。

```bash
#!/bin/bash

NAME=$1

if [ -z "$NAME" ]; then
  echo "第一引数としてプロファイル名を指定してください。"
  exit 1
fi

cat << EOF >> "./provider.tf"

provider "aws" {
  alias   = "$NAME"
  region  = "us-east-1"
  profile = "$NAME"
  default_tags {
    tags = local.common_tags
  }
}
EOF

echo "provider.tfに設定を追加しました。"

cat << EOF >> "./main.tf"

module "$NAME" {
  source                 = "./modules"
  destination_bucket_arn = data.terraform_remote_state.cur_setup_destination.outputs.destination_bucket_arn
  providers = {
    aws = aws.$NAME
  }
}
EOF

echo "main.tfに設定を追加しました。"
```

---

以上で、この記事は終了です。
次回の記事では、[Step3:データソース、およびダッシュボードの作成](https://qiita.com/masatomasato1224/items/311e890ade9b48700cbe#%E6%A7%8B%E7%AF%89%E3%82%B9%E3%83%86%E3%83%83%E3%83%97:~:text=%E3%83%87%E3%83%BC%E3%82%BF%E3%82%BD%E3%83%BC%E3%82%B9%E3%80%81%E3%81%8A%E3%82%88%E3%81%B3%E3%83%80%E3%83%83%E3%82%B7%E3%83%A5%E3%83%9C%E3%83%BC%E3%83%89%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B)について解説します。
※ 現在、執筆中です。

それでは、次回の記事でお会いしましょう。
