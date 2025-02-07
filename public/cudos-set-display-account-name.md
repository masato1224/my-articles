---
title: 'CUDOS構築記 番外編: アカウント名を表示する'
tags:
  - AWS
  - Terraform
  - CFM
  - CUDOS
  - CloudIntelligenceDashboards
private: false
updated_at: ''
id: null
organization_url_name: ateam-life-design
slide: false
ignorePublish: false
---

こちらの記事は、[CUDOS構築記 Vol.3: CURの出力と集約先へのレプリケーション設定をする](https://qiita.com/masatomasato1224/items/cce0ec2a3e8aa039981d)の続きです。
前回で無事ダッシュボードの作成が完了しました。今回は番外編としてダッシュボードに表示されるアカウントの名称を変更する方法を紹介します。

## アカウントの名称を変更とは

CUDOSを構築した場合、デフォルトではアカウントIDが表示されます。

![2025-02-07-11-18-42.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/ccee05f7-2f32-4fa3-cd52-453f93432428.png)

アカウント数が少ない場合は問題になりませんが、アカウント数が増えるとアカウントIDだけではどのアカウントか分かりにくくなります。
そのため、アカウントIDの代わりにアカウント名を表示するように変更します。

## アカウント名を表示する方法

[公式ドキュメント](https://catalog.workshops.aws/awscid/en-US/dashboards/foundational/cudos-cid-kpi/add-accounts)には3つの方法が記載されています。

1. AWS Organizationsのアカウントマッピングを活用する
1. AWS Cost Categoryを利用してアカウント名を追加する
1. アカウントマップマッピングデータを用意する

今回は3の方法について解説します。

## おおまかな手順

1. アカウントIDとアカウント名のマッピングデータの作成
2. マッピングデータをS3にアップロード
3. Athenaにマッピングtableを作成
4. Athenaのマッピング用Viewを更新
5. QuickSightの設定変更
6. IAMRolePolicyの修正
7. SummaryViewを更新

## アカウントIDとアカウント名のマッピングデータの作成

公式ドキュメント内にテンプレートのダウンロードリンクがあるので、テンプレートをダウンロードします。
![2025-02-07-11-35-21.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/5a0b755c-0c34-a4b2-0c9e-96aae38f1ed2.png)

```csv: テンプレート
account_id,account_name,business_unit,team,cost_center
1.23457E+11,Master,CloudOps,core,1234
1.23457E+11,Prod,App,Demo,5678
```

`account_id`,`account_name`以外は任意のColumnになるので今回は削除します。
account_nameにそれぞれのアカウント名を記載します。

```csv: マッピングデータ
account_id,account_name
1111111111,【開発G1】DevAccount
2222222222,【開発G1】StagingAccount
3333333333,【開発G1】ProdAccount
4444444444,【開発G2】認証基盤
5555555555,【開発G2】サブシステムA
```

## マッピングデータをS3にアップロード

手動でアップロードしてもいいですが、運用が楽になるようにterraformで管理します。

```tf
resource "aws_s3_bucket" "account_mapping" {
  bucket = "cudos-account-map"
}

// ... 省略 ...
// aws_s3_bucket_ownership_controls
// aws_s3_bucket_public_access_block
// etc

resource "aws_s3_object" "setting_file" {

  key          = "account-map/account_map.csv"
  bucket       = aws_s3_bucket.account_mapping.bucket
  source       = "./account_map.csv"
  content_type = "text/csv"
  etag         = filemd5("./account_map.csv")
}
```

```bash
terraform apply
```

## Athenaにマッピングtableを作成

Athenaでマッピングデータを参照するためのtableを作成します。

```sql
CREATE EXTERNAL TABLE `account_mapping`(
  `account_id` string,
  `account_name` string,
)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY ','
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://cudos-account-map/account-map/'
TBLPROPERTIES (
  'has_encrypted_data'='false',
  'skip.header.line.count'='1'
)
```

## Athenaのマッピング用Viewを更新

マッピング用Viewを更新して、アカウント名を表示できるようにします。

```sql
CREATE OR REPLACE VIEW account_map AS
SELECT DISTINCT
a.line_item_usage_account_id "account_id"
, b.account_name
FROM
  ((
  SELECT DISTINCT
   line_item_usage_account_id
  FROM
   customer_cur_data.cur
) a
LEFT JOIN (
  SELECT DISTINCT
    "lpad"("account_id", 12, '0') "account_id"
    , account_name
  FROM
    customer_cur_data.account_mapping
) b ON (b.account_id = a.line_item_usage_account_id))
```

## QuickSightの設定変更

マッピングデータを保持しているs3へのアクセス権限を追加します。

- QuickSightの管理コンソールから、`セキュリティとアクセス許可` へアクセス
- アカウントマッピングを保持しているS3バケットにアクセス権限を追加

## IAMRolePolicyの修正

次のIAMRoleに対して、S3バケットへのアクセス権限を追加します。

- Role: `CidCmdQuickSightDataSourceRole`
  - `s3:ListBucket`
  - `s3:GetObject`
  - `s3:GetObjectVersion`

```json
        {
            "Sid": "AllowListBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                ...省略...
                "arn:aws:s3:::cudos-account-map" // 追加
            ]
        },
        {
            "Sid": "AllowReadBucket",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": [
                ...省略...
                "arn:aws:s3:::cudos-account-map/*" // 追加
            ]
        }
```

## SummaryViewを更新

QuickSightでSummaryViewを更新します。1分から2分程度で更新されます。

- ![2025-02-07-12-27-04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/3e2f2a52-076e-471f-681b-013cbc6ebf34.png)
- ![2025-02-07-12-27-47.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/9b51b64b-f9ad-29fa-cbff-3362b04a8cec.png)
- ![2025-02-07-12-28-18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/a17490a3-c40c-3ce4-d991-c3835e5a6dd7.png)

## 結果確認

更新が終わったら、ダッシュボードにアクセスしてアカウント名が表示されるようになっていれば成功です。
