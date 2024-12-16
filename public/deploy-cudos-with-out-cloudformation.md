---
title: 'CUDOS構築記 Vol.1: CUDOSの概要と構築方法の検討'
tags:
  - AWS
  - Terraform
  - CFM
  - CUDOS
  - CloudIntelligenceDashboards
private: false
updated_at: '2024-12-16T12:02:25+09:00'
id: 311e890ade9b48700cbe
organization_url_name: ateam-life-design
slide: false
ignorePublish: false
---

この記事は、[Ateam LifeDesign Advent Calendar 2024](https://qiita.com/advent-calendar/2024/ateam-life-design) 16日目の記事になります。

## はじめに

こんにちは、エイチームライフデザインの鈴木です。
私は基盤技術グループという組織横断型チームに所属しており、AWSコスト最適化に取り組んでいます。

AWSコスト最適化の出発点として、**`CUDOS`**（Cost and Usage Dashboards Operations Solution）と呼ばれるコストダッシュボードを構築しました。
これにより、エイチームライフデザインが保有するすべてのAWSアカウントのコスト情報を一元的に可視化し、コスト最適化の施策を検討できるようになりました。

CUDOSの導入過程を複数回に分けて記事にまとめました。
本記事では、まずCUDOSの概要と構築時の検討内容を紹介します。

## 本記事の構成

- CUDOSとは何か
- CUDOSのしくみ
- 構築ステップ
- 構築方法の検討

## CUDOSとは何か

複数のAWSアカウントのコスト情報を集約して、可視化&分析するためのダッシュボードです。

https://aws.amazon.com/jp/blogs/news/aws-costdashboard-usecases-cud-cudos/

コスト分析には`Cost Explorer`を用いることが一般的ですが、CUDOSを利用することで、より詳細かつ容易にコスト分析が可能になります。CUDOSには主に、次のような特徴があります。

- 複数アカウントのコスト情報を集約して可視化できる
- リソースレベルのコスト情報を可視化できる
- Daily/Weekly/Monthlyのコスト推移を確認できるUIがデフォルトで用意されている

こちらの[デモサイト](https://d1s0yx3p3y3rah.cloudfront.net/anonymous-embed?dashboard=cudos)をご覧いただくと、CUDOSの機能を体験できます。

## CUDOSのしくみ

- 各アカウントからCUR（Cost & Usage Report）を収集する
- CURを加工してデータソースを作成する
- QuickSightを使ってデータソースを可視化する

![2024-12-13-16-09-26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/0cfd83cc-4ec1-b7a3-3be0-bc67853bd81f.png)

*画像はhttps://catalog.workshops.aws/awscid/en-US/dashboards/foundational/cudos-cid-kpi から引用*

## 構築ステップ

CUDOSの構築には、大きく分けて3つのステップがあります。

1. CURの集約先を作成する
2. CURの出力と集約先へのレプリケーション設定をする
3. データソース、およびダッシュボードを作成する

ステップ1とステップ3はダッシュボードを設置するアカウントが対象となりますが、ステップ2は可視化対象アカウントがすべて対象となります。

![2024-12-13-17-28-10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/7a5b994a-d825-3644-b66c-c4e4ecd6152a.png)

## 構築方法の検討

CUDOSにはそれぞれのステップに対して複数の構築方法が用意されています。

| NO  | ステップ1               | ステップ2               | ステップ3               |
| --- | ----------------------- | ----------------------- | ----------------------- |
| 1   | CloudFormation Template | CloudFormation Template | CloudFormation Template |
| 2   | CloudFormation Template | CloudFormation Template | CLI（cid-cmd tool）     |
| 3   | Terraform Module        | Terraform Module        | Terraform Module        |
| 4   | Terraform Module        | Terraform Module        | CLI（cid-cmd tool）     |

公式サイトでは[NO.1 CloudFormation Templateを使った構築](https://catalog.workshops.aws/awscid/en-US/dashboards/foundational/cudos-cid-kpi/deploy)が解説されています。
しかし、検討を重ねた結果、最終的にはNO.4 Terraform Module + CLI（cid-cmd tool）を選択しました。

![2024-12-13-17-33-21.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/444225/47b8ea84-0cc5-a80e-e875-af4e12baab62.png)

### なぜTerraform Module + CLI（cid-cmd tool）を選択したのか

構築・運用フェーズともにオペレーションコストがもっとも低いと判断したからです。

まず、公式で解説されているNO.1を検討しましたが、オペレーションコストが高すぎました。
公式サイトの解説では、アカウントのコンソールにログインして手動でのオペレーションが前提となっていました。そして、ステップ2のオペレーションは可視化対象アカウントがすべて関与します。また、アカウントが追加される際にも同様のオペレーションが必要となります。そのため、アカウント数が増えるにつれてオペレーションコストが上昇するNO.1は選択肢から除外され、同様にCloudFormation Templateを利用するNO.2も選択肢から外されました。

残る選択肢は、NO.3とNO.4です。
NO.3はすべてのステップでTerraform Moduleを利用する方法です。一方、NO.4はステップ1、2でTerraform Moduleを利用し、ステップ3でcid-cmd toolを利用する方法です。一見するとNO.3の方がシンプルです。しかし、[CUDOS Frameworkのリポジトリ](https://github.com/aws-samples/aws-cudos-framework-deployment?tab=readme-ov-file#welcome-to-cloud-intelligence-dashboards-cudos-framework-automation-repository)の調査を進めたところステップ3のTerraform ModuleはCloudFormation Templateのラッパーであることが分かりました[^1]。そして、CloudFormation Templateもcid-cmd toolを内部で呼び出していました。また、NO.3に関しては未解決の[issue](https://github.com/aws-samples/aws-cudos-framework-deployment/issues/1029)があったため、NO.4の方が安定していると判断しました。

[^1]: `Support for Terraform-only Deployment`という[issue](https://github.com/aws-samples/aws-cudos-framework-deployment/issues/725)が立っているので、将来的にはプレーンなHCLのみに変更される可能性があります。

---

以上で、この記事は終了です。
本記事ではCUDOSの概要と構築方法の検討について紹介しました。いかがでしたでしょうか。
次回の記事では、CUDOSの構築ステップを具体的な実装を交えて紹介します。
※ 現在、執筆中です。

それでは、次回の記事でお会いしましょう。
