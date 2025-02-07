---
title: 'CUDOS構築記 Vol.4: データソース、およびダッシュボードを作成する'
tags:
  - AWS
  - Terraform
  - CFM
  - CUDOS
  - CloudIntelligenceDashboards
private: false
updated_at: '2025-02-03T14:16:46+09:00'
id: cce0ec2a3e8aa039981d
organization_url_name: ateam-life-design
slide: false
ignorePublish: false
---

こちらの記事は、[CUDOS構築記 Vol.3: CURの出力と集約先へのレプリケーション設定をする](https://qiita.com/masatomasato1224/items/998e89859aeed4db8c66)の続きです。
前回は、CURの集約先の作成について紹介しました。今回は、[Step3:データソース、およびダッシュボードの作成](https://qiita.com/masatomasato1224/items/311e890ade9b48700cbe#%E6%A7%8B%E7%AF%89%E3%82%B9%E3%83%86%E3%83%83%E3%83%97:~:text=%E3%83%87%E3%83%BC%E3%82%BF%E3%82%BD%E3%83%BC%E3%82%B9%E3%80%81%E3%81%8A%E3%82%88%E3%81%B3%E3%83%80%E3%83%83%E3%82%B7%E3%83%A5%E3%83%9C%E3%83%BC%E3%83%89%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B)についてコードを交えながら解説します。

## 本記事の構成

- 前提のおさらい
- 構築範囲の確認
- 構築手順

## 前提のおさらい

- CUDSの構築ステップは3つ
  - （作成済み）Step1: CURの集約先を作成する
  - （作成済み）Step2: CURの出力と集約先へのレプリケーション設定をする
  - Step3: データソース、およびダッシュボードを作成する
- 構築先アカウント
  - Step1,3:ダッシュボードを設置するアカウント
  - Step2: 可視化対象アカウントすべて
- CUDOSの構築方針
  - Step1,2はTerraformで構築
  - Step3はCLI（cid-cmd tool）で構築

## 構築範囲の確認

![構築範囲](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/87d62f16-37a5-d69a-d77d-b5463e8d7ba9.png)

赤枠で囲ってある範囲が、本記事で構築する対象です。
データソースの設定とダッシュボードを作成します。
ここまでの構築が終わっていれば、コマンドを叩くだけでダッシュボードが作成されます。

## 構築作業

### 前提

- Step2の構築が完了していること
- 設定先のAWSアカウントに対する適切なアクセス権限を保持していること
- `Athena`の`workgroup`が設定済みであること
  - 未設定の場合は、サンプルコードを参考に設定してください
- `QuickSight`の`Enterprise edition`が利用可能であること
  - QuickSightを利用したことがない場合は、[公式ドキュメント 3.1-prepare-amazon-quicksight](https://catalog.workshops.aws/awscid/en-US/dashboards/foundational/cudos-cid-kpi/deploy#3.1-prepare-amazon-quicksight)を参考に設定してください

#### Athenaのworkgroupの設定サンプル

```ts: Athenaのworkgroupの設定サンプル
resource "aws_athena_workgroup" "cid" {
  name        = "CID"
  description = "Used for CloudIntelligenceDashboards"

  configuration {
    enforce_workgroup_configuration    = true
    publish_cloudwatch_metrics_enabled = true

    result_configuration {
      output_location = "s3://<your-bucket-name-of-athena-result>/"

      encryption_configuration {
        encryption_option = "SSE_S3"
      }
    }
  }
}
```

### 事前準備

ダッシュボードを設置するAWSアカウントにログインして、`CloudShell`を起動します。
この際に、リージョンが`us-east-1`であることを確認してください。

### CILのインストール

コマンドラインツールである`cid-cmd`をインストールします。

```bash
python3 -m ensurepip --upgrade
...
pip3 install --upgrade cid-cmd
...
```

### デプロイ

```bash
cid-cmd deploy
```

インタラクティブなプロンプトが表示されるので、入力していきます。

```bash
# デプロイするダッシュボード: CUDOS Dashboard v5
? [dashboard-id] Please select a dashboard to deploy: (Use arrow keys)
   FOUNDATIONAL
 »   [cudos-v5] CUDOS Dashboard v5

# Athenaのworkgroup: 事前に作成したものを選択
? [athena-workgroup] Select Amazon Athena workgroup to use: (Use arrow keys)
 » CID
   primary

# データソース: 新規作成
? [quicksight-datasource-id] Please choose DataSource (Select the first one if not sure): (Use arrow keys)
 » CID-CMD-Athena <CREATE NEW DATASOURCE>

# QuickSightのロール: 新規作成
? [quicksight-datasource-role] Please choose a QuickSight role. It must have access to Athena: (Use arrow keys)
   <USE DEFAULT QuickSight ROLE (You will need to login to QuickSight (https://quicksight.aws.amazon.com/sn/admin#aws) and configure S3 and Athena access there)>
   aws-quicksight-service-role-v0
 » CidCmdQuickSightDataSourceRole <ADD NEW ROLE>

# Athenaのデータベース: 新規作成
? [athena-database] Select AWS Athena database to use: (Use arrow keys)
 » customer_cur_data (CREATE NEW)

# Step1で作成したCURの集約先のバケットを選択
? [view-cur-s3-bucket] Enter a bucket with CUR: (Use arrow keys)
 » cid-xxxxxxx-shared

# CURのパス
? [view-cur-location] Enter an S3 path. We support only 2 types of CUR path: s3://{bucket}/cur and s3://{bucket}/{prefix}/{name}/{name}: s3://cid-xxxxxxx-shared/cur/

# GlueCrawlerのロール: 新規作成
? [crawler-role] Provide a crawler role name: (Use arrow keys)
 » CidCmdCurCrawlerRole <CREATE NEW>

# ダッシュボード上のアカウント表示名の設定
# こちらは後から変更するので、Dummyを選択
? [account-map-source] Please select account metadata collection method: (Use arrow keys)
 » Dummy (CUR account data, no names)
   AWS Organizations (one time account listing)
   CSV file (relative path required)

# Timezoneの設定
? [timezone] Please select timezone for datasets scheduled refresh.: Asia/Tokyo

# ダッシュボードの共有設定: こちらはお好みで
 [share-with-account] Share this dashboard with everyone in the account?: (Use arrow keys)
 » No

CUDOS Dashboard v5 is available at: <URL>
```

最後に出力されるたURLへアクセスして、ダッシュボードが表示されることを確認してください。

![dashboard](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/00c160e8-fe5f-adf8-3888-c2290a132710.png)

CURに関しては反映まで24時間ほどかかるので、時間をおいて確認してください。

---

以上で、この記事は終了です。
これまで4回にわたって、CUDOSの構築手順を解説してきました。
CUDOSを構築するに当たり、情報が少なく手探りで進めることが多かったので、今後の誰かの参考になればと思い執筆しました。
もし、何か不明点や誤りがあれば、お気軽にコメントいただけると幸いです。
