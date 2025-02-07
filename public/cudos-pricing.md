---
title: 'CUDOS構築記 番外編②: CUDOSの運用費用'
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

これまでCUDOSについての記事を書いてきましたが、今回は番外編としてCUDOSの運用費用について説明します。

過去の記事はこちらをご覧ください。

- [CUDOS構築記 Vol.1: CUDOSの概要と構築方法の検討](https://qiita.com/masatomasato1224/items/311e890ade9b48700cbe)
- [CUDOS構築記 Vol.2: CURの集約先を作成する](https://qiita.com/masatomasato1224/items/015b4c9f5bdeaeeb5cf2)
- [CUDOS構築記 Vol.3: CURの出力と集約先へのレプリケーション設定をする](https://qiita.com/masatomasato1224/items/998e89859aeed4db8c66)
- [CUDOS構築記 Vol.4: データソース、およびダッシュボードを作成する](https://qiita.com/masatomasato1224/items/cce0ec2a3e8aa039981d)
- [CUDOS構築記 番外編: アカウント名を表示する](https://qiita.com/masatomasato1224/items/40a16048d1c35eeede04)

## 大体いくらかかるのか

ざっくりとした金額ですが、CUDOSの運用費用は1ヶ月あたり100-200ドル程度になります。

以下は[公式サイト](https://catalog.workshops.aws/awscid/en-US/faqs#pricing)のFAQsからの引用です。

> ･ Total: $100-$200 monthly

ただ、これには色々と条件があり実際の金額は環境によって異なります。
もう少し掘り下げていきます。

## 費用の要素

まず、CUDOSの費用は次の要素によって決まります。

- QuickSightのユーザ数
- 連携するCURのデータサイズ

そして、CUDOSの費用は5つに分けられます。

- QuickSight
  - ユーザ費用
  - SPICE費用
- S3費用
- Athena費用
- Glue費用

関係を図示すると次のようになります。

![2025-02-07-18-27-18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/1ca83e2e-5b9a-0f2f-cb25-bce30e6357dc.png)

## 費用にもっとも影響する要素はなにか

月々のAWS費用が10万ドル程度までであれば、QuickSightのユーザ費用がもっとも費用に影響します。
※ CURのデータ量はリソース数の影響を大きく受けます、金額はあくまで目安です。
以下は弊社の実績ですが、QuickSightのユーザ費用が大部分を締めていることがわかります。

![2025-02-07-17-51-16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/bf791d76-6f7e-5d3b-54b1-fa8d8b6062c5.png)
![2025-02-07-17-52-20.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/9470680f-70aa-7a9d-ca14-aefb04dcb0e9.png)

## どうやって見積もるのか

ここまでを踏まえ得て、ざっくりとした見積もり方法を説明します。

### 月々のAWS費用が10万ドル程度

- QuickSightのユーザ費用 + 10〜20ドル（S3, Athena, Glue費用 ※ バッファ込）

### 月々のAWS費用が10万ドルを大きく超える場合

- ①もっとも高額なアカウントのCURを出力して、データサイズを確認する
- ②データサイズを元にS3, Athena, Glue, SPICEの費用を見積もる → 基準値
  - ※ 弊社の実績値では、s3の費用と比較したAthena、Glue、SPICEの費用は次のようになっています
    - Athena: s3費用の約19%
    - Glue: s3費用の約53%
    - SPICE: s3費用の約95%
- ③基準値を元にその他アカウントのデータサイズと費用を算出する
- ②と③にQuickSightのユーザ費用を加える

## おまけ（やらかした話）

QuickサイトにはAmazon Qを利用できるProユーザがあります。
Proユーザが1人でもいると、250 USD/月のAmazon Qの有効化料金が別途発生します。

私はユーザを追加する際に誤ってProユーザを追加してしまい、有効化料金が発生させてしまったことがあります。幸いにも月末が近かったので全額請求されることはありませんでしたが、請求書を見てびっくりしました。ユーザ追加の際は、必ずProユーザでないことを確認しましょう。
