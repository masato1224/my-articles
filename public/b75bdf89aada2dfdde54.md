---
title: AWSセキュリティグループをGithub Actionsから動的に更新･削除する
tags:
  - AWS
  - GitHubActions
private: false
updated_at: '2023-12-21T13:21:38+09:00'
id: b75bdf89aada2dfdde54
organization_url_name: ateam-life-design
slide: false
ignorePublish: false
---
この記事は[Ateam LifeDesign Advent Calendar 2023](https://qiita.com/advent-calendar/2023/ateam-life-design) 2023の21日目記事です

# 背景

今年、弊社ではコード管理システムの移行という大きな意思決定を行いました。当初は「リポジトリを移すだけ」と思っていました｡ しかし実際に移行作業を進めた結果、予期せぬ課題がいくつか浮上しました。今回は、その中の一つに焦点を当て、解決に向けたアプローチを共有したいと思います。

## 前提

- これまではGitLabをセルフホステッド運用していた(通称「`社内GitLab`」)
- コード管理システムの候補は`GitLab SaaS版`、`GitHub`の2つ
- 移行先の選択と移行スケジュールは､各プロダクトチームで意思決定を行う
- 私が所属するチームでは`GitHub`を選択した

## 浮上した課題

GitHubのCI/CDランナーから社内GitLabへアクセスを許可することが必要

- 社内GitLabは、弊社のAWSネットワーク上でホスティングされており、IPアドレスに基づくアクセス制御が行われていた
- 一部の共通社内パッケージは、社内GitLabで管理・配布されている状況

## 打ち手の検討と比較

以下の考慮事項を踏まえ､いくつかの打ち手を検討しました

### 検討時の考慮事項

+ プロダクトの方針として社内パッケージの依存を減らしていきたい
+ プロダクト内で影響を受けるリポジトリは1つのみ
  + とはいえプロダクトにおける主要リポジトリであり､移行の対象から外すことはできない
+ Github移行を決めた他のプロダクトでも同様の課題が発生する
  + しかしながら､影響を受けるリポジトリの数は多くない

### 打ち手の一覧

1. 社内GitLabのIPアドレス制御の解除
2. 共通社内パッケージのGitHubへの移行
3. Self-hosted Runnerの用意と固定IPの割り当て
4. GitHubのIPアドレスレンジを事前にホワイトリストに登録
5. 実行時にGitHub RunnerのIPアドレスを取得し、ホワイトリストに動的に追加

### 検討内容

#### 1. 社内GitLabのIPアドレス制御の解除

- ユーザ認証を行っているので、IPアドレス制御解除も選択肢たりえる
- 個人の判断では決裁できない

#### 2. 共通社内パッケージのGitHub移行

- 共通社内パッケージの移行先や時期が未定
  - 共通社内パッケージの移行意思決定は他プロダクトチームの管轄
- 対象パッケージが10以上ある
    - 仮に意思決定がなされたとしても移行を待つのは現実的でない

#### 3. Self-hosted Runnerの用意と固定IPの割り当て

- Runnerの構築とメンテナンスコストを考えると、積極的に採用したくはない

#### 4. CI/CDランナーのIPアドレスレンジを事前にホワイトリストに登録

- ホワイトリストの管理について課題が浮上する
  - IPアドレスとレンジが広い
  - 公式ドキュメントでも変更の可能性が言及されている
    - 管理できない場合､複数プロダクトのCI/CDが突然失敗することになる

#### 5. 実行時にCI/CDランナーのIPアドレスを取得し、ホワイトリストに動的に追加

- Github Actionsのワークフローで完結させることができる
  - 構築コスト･破棄コストが低い
  - メンテナンスコストはリポジトリの数に依存する
- ホワイトリストの管理が容易


## 検討結果

`5. 実行時にCI/CDランナーのIPアドレスを取得し、ホワイトリストに動的に追加`の手法を採用しました

+ 決め手
  + 導入ハードルが低くスピード感を持って対応できること
    + 構築コストが低く､自チームだけで意思決定を行える
    + 破棄が用意に行える
      + 後に`社内GitLabのIPアドレス制御の解除`,`通社内パッケージのGitHub移行`が決まったとしてもワークフローの修正のみで対応可能
  + 目立ったデメリットがない
    - リポジトリ数の増加に伴うメンテナンスコストについては、以下の理由で許容できると判断
      - 現時点で対象となるリポジトリが少ない
      - 今後増える可能性も低い


## Github Actionsのワークフロー

### 概要

1. CI/CDランナーのIPアドレスを取得
2. セキュリティグループのイングレスルールに取得したIPアドレスを追加
3. アプリケーションのビルドとデプロイを実行
4. セキュリティグループのイングレスルールから取得したIPアドレスを削除
   + ポイント：
     + `always()`を指定し、ステップが失敗した場合でも実行されるようにする
     + アプリケーションのビルドとデプロイ後に再度AWS認証を行う
       + ※ アプリケーションのデプロイ先が別のAWSアカウントに存在するため


#### コード

<details>
<summary>折りたたみ</summary>

```yaml
name: Build and Deploy

on:
  workflow_call:
    inputs:
      aws_region:
        required: true
        type: string
      aws_role:
        required: true
        type: string

jobs:
  build_and_push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Get current runner IP
        id: get_ip
        run: |
          echo "ip=$(curl -s https://ipecho.net/plain)" >> $GITHUB_OUTPUT

      - name: Configure AWS credentials for permit ip address
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-region: ${{ inputs.aws_region }}
          role-to-assume: ${{ inputs.aws_role }}

      - name: Add Ingress Rule
        run: |
          aws ec2 authorize-security-group-ingress \
            --group-id sg-xxxxxxx \
            --protocol tcp \
            --port 22 \
            --cidr "${{ steps.get_ip.outputs.ip }}/32"

      # アプリケーションのbuildとデプロイstep設定は省略

      - name: configure aws credentials for unpermit ip address
        # アプリケーションのデプロイ先が別のAWSアカウントに存在するので､アカウントを切り替えるために再度ログイン
        if: always()
        uses: aws-actions/configure-aws-credentials@v3
        with:
          aws-region: ${{ inputs.aws_region }}
          role-to-assume: ${{ inputs.aws_role }}

      - name: remove Ingress Rule
        if: always()
        run: |
          aws ec2 revoke-security-group-ingress \
            --group-id sg-xxxxxxx \
            --protocol tcp \
            --port 22 \
            --cidr "${{ steps.get_ip.outputs.ip }}/32"

```

</details>


## 実際に運用してみて

現在のところ、特に大きな問題は発生していません

- IPアドレスが重複しエラーが発生するケースがありましたが、基本的には再実行で解決可能でした
- さらに、`concurrency`を設定することでエラー発生頻度を抑制できました
- ただし、運用は現状1リポジトリのみであり、リポジトリ数が増えた場合の影響はまだ未知数です

## 蛇足

今回の構築にあたり、AWSのセキュリティグループをTerraformによるインフラストラクチャー as コード（IaC）で実装し、またAWS認証にはOpenID Connect（OIDC）プロバイダーを用いました。これらの詳細な情報については、別の機会に紹介したいと思います。
