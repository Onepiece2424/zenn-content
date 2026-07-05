---
title: "GitHub Actions × OIDC で実現するセキュアなCI/CDパイプラインの作成"
emoji: "😄"
type: "tech"
topics:
  - "aws"
  - "githubactions"
  - "terraform"
  - "oidc"
published: true
published_at: "2026-07-04 12:26"
---

# はじめに

ポートフォリオサイトを AWS 上にホスティングするにあたり、「毎回手動で `aws s3 sync` を叩くのは面倒だし、IAM ユーザーの長期クレデンシャルをシークレットに持たせたくない」という課題がありました。

本記事では、**GitHub Actions + OIDC** を使ってアクセスキーを一切持たずに S3 デプロイ → CloudFront キャッシュ無効化までを自動化した実装を解説します。インフラは全て Terraform で管理しています。

GitHub Actions の認証部分をOIDCで実装しようとしている方の参考になれば幸いです。


# システム構成の概要

今回のポートフォリオは以下の構成になっています。

![](https://static.zenn.studio/user-upload/aaac7b1f2c6d-20260705.png)


```
ブラウザ
  └─ CloudFront
       ├─ /api/* → API Gateway → Lambda → Amazon Bedrock (Nova Pro)
       └─ /*     → S3 (静的サイト)
```

フロントエンドは `index.html` 一枚の静的サイトで、S3 に配置したものを CloudFront 経由で配信しています。この静的ファイルのデプロイを自動化するのが今回のCI/CDです。

# OIDC とは
GitHubにおけるOIDC（OpenID Connect）とは、ID/パスワードや長期的なアクセスキーの代わりに、「一時的な認証情報」を使ってAWSやAzureなどのクラウドサービスへ安全にアクセスするための仕組みになります。

https://docs.github.com/ja/actions/concepts/security/openid-connect



# なぜ OIDC か？

従来のやり方だと、IAM ユーザーを作成してアクセスキー（`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`）を GitHub Secrets に登録する方法が一般的でした。しかしこの方法にはいくつかのリスクがあります。

- 長期クレデンシャルが漏洩した場合のリスクが高い
- キーのローテーションを手動で管理する必要がある
- 最小権限の原則から外れやすい

**OIDC（OpenID Connect）** を使うと、GitHub Actions が実行される際にAWSへの短期トークン（一時的なクレデンシャル）を動的に発行できます。アクセスキーの発行・管理が完全に不要になり、セキュリティが大幅に向上します。

```
GitHub Actions ワークフロー
  │
  ├─ 1. OIDC トークンを GitHub から取得
  ├─ 2. AWS STS に AssumeRoleWithWebIdentity を要求
  ├─ 3. 一時クレデンシャル（15分〜1時間）を取得
  └─ 4. S3 / CloudFront API を実行
```

https://docs.github.com/ja/actions/concepts/security/openid-connect#benefits-of-using-oidc


# Terraform でのインフラ実装

OIDC 連携に必要な AWS 側のリソースを `github_oidc` モジュールとして実装しました（今回作成したコードはこのブログ記事の一番下に掲載しております）。

https://registry.terraform.io/providers/-/aws/latest/docs/resources/iam_openid_connect_provider

### 1. OIDC プロバイダーの登録

```hcl:terraform/modules/github_oidc/main.tf

resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  # thumbprint_listにダミー値を設定
  thumbprint_list = [
    "ffffffffffffffffffffffffffffffffffffffff"
  ]

  tags = {
    Name        = "github-actions-oidc"
    Environment = var.env
  }
}
```

上記では、AWS アカウントに GitHub の OIDC プロバイダーを登録しています。
`thumbprint_list` は、主にAWSなどのクラウド環境でOpenID Connect (OIDC) 連携を行う際、外部の認証プロバイダー（GitHubなど）のサーバー証明書の正当性を検証するために指定するサーバー証明書フィンガープリント（拇印）のリストであり、以前は GitHub の公式値を使用しておりました。

https://github.blog/changelog/2023-06-27-github-actions-update-on-oidc-integration-with-aws/

https://dev.classmethod.jp/articles/aws-cdk-github-oidc-multiple-thumbprints/

ただ、GitHub 側の証明書が更新された時、ユーザー側で `thumbprint_list` をその都度更新する必要があり、少し面倒でした。

そこで、いろいろ調べてみると、`token.actions.githubusercontent.com`を認証する際の設定の時、`thumbprint_list` に関しては、`f`のダミー値で良いようです。

https://github.com/aws-actions/configure-aws-credentials/pull/764/changes#diff-b335630551682c19a781afebcf4d07bf978fb1f8ac04c6bf87428ed5106870f5R223-R275

https://wandfuldays.com/articles/c7cx_y8_ycc/

そこで、実際に`thumbprint_list` に`f`のダミー値を設定し、ワークフローを実行したところ、うまくパイプラインを通すことができました。



### 2. IAM ロールと信頼ポリシー

```hcl:terraform/modules/github_oidc/main.tf
resource "aws_iam_role" "github_actions" {
  name               = "${var.project_name}-github-actions-role-${var.env}"
  assume_role_policy = data.aws_iam_policy_document.github_actions_assume_role.json
}

data "aws_iam_policy_document" "github_actions_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    # リポジトリ単位で Assume を制限
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${var.github_repo}:*"]
    }
  }
}
```

上記の`assume_role_policy` は、今回の場合「GitHub Actions がこの IAM ロールを引き受け（Assume）てもよい条件」を定義しているポリシーになります。

ポイントは 2 つの `condition` ブロックです。

- `aud` 条件：AWS STS への要求であることを確認
- `sub` 条件：**指定したリポジトリからのリクエストのみ**許可（他のリポジトリからの不正な Assume を防ぐ）

https://developer.hashicorp.com/terraform/language/expressions/conditionals


### 3. デプロイに必要な最小権限ポリシー

```hcl:terraform/modules/github_oidc/main.tf
data "aws_iam_policy_document" "github_actions_deploy" {
  # S3 への同期に必要な権限
  statement {
    effect = "Allow"
    actions = [
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:GetObject",
      "s3:ListBucket",
    ]
    resources = [
      "arn:aws:s3:::${var.s3_bucket_name}",
      "arn:aws:s3:::${var.s3_bucket_name}/*",
    ]
  }

  # CloudFront キャッシュ無効化
  statement {
    effect  = "Allow"
    actions = ["cloudfront:CreateInvalidation"]
    resources = [
      "arn:aws:cloudfront::${var.aws_account_id}:distribution/${var.cloudfront_distribution_id}",
    ]
  }
}
```

上記では、デプロイに必要な権限のみを付与しています。
`s3:*` や `cloudfront:*` のようなワイルドカードは使わず、アクション・リソースを両方で絞るのが重要です。

### 4. モジュールの呼び出し

```hcl:terraform/environments/dev/main.tf

module "github_oidc" {
  source = "../../modules/github_oidc"

  env                        = var.env
  aws_region                 = var.aws_region
  aws_account_id             = data.aws_caller_identity.current.account_id
  project_name               = "bedrock-ai-portfolio"
  github_repo                = var.github_repo
  s3_bucket_name             = "frontend-static-content-bucket-${var.env}-${var.bucket_suffix}"
  cloudfront_distribution_id = module.s3_cloudfront.cloudfront_distribution_id
}
```

`terraform apply` 後、IAM ロールの ARN が output に出力されます。

```hcl:terraform/environments/dev/outputs.tf
output "github_actions_role_arn" {
  value       = module.github_oidc.role_arn
  description = "GitHub Secrets の AWS_ROLE_ARN に設定する IAM ロール ARN"
}
```

この ARN を GitHub Secrets の `AWS_ROLE_ARN` に設定すれば準備完了です。


## GitHub Actions ワークフロー

```yaml:.github/workflows/deploy.yml

name: Deploy to S3 and Invalidate CloudFront

on:
  pull_request:
    types:
      - opened
      - synchronize
      - reopened

permissions:
  id-token: write   # OIDC トークン取得に必要
  contents: read    # リポジトリのチェックアウトに必要

jobs:
  deploy:
    name: Deploy Static Site
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Deploy static files to S3
        run: |
          aws s3 sync . s3://${{ secrets.S3_BUCKET_NAME }} \
            --exclude "*" \
            --include "index.html" \
            --include "images/*" \
            --delete

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

### ワークフローのポイント

**①`permissions` の明示的な設定**
`id-token: write` を付けないと OIDC トークンが取得できません。GitHub Actions ではデフォルトで OIDC トークンへのアクセスが制限されているため、明示的に指定が必要です。

https://docs.github.com/ja/actions/reference/workflows-and-actions/workflow-syntax#permissions

**②`aws-actions/configure-aws-credentials@v4`**
このアクションが OIDC トークンの取得から `AssumeRoleWithWebIdentity`（外部サービスの身分証明書（OIDCトークン）を使って、一時的にAWSのIAMロールを借りる仕組み） の呼び出しまでを全て行ってくれます。後続のステップでは通常の `aws` CLI コマンドがそのまま使えます。

https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_credentials_temp_control-access_assumerole.html

**③`aws s3 sync` の `--exclude` / `--include`**
`aws s3 sync`とは、コピー元とコピー先の差分（新規・更新ファイル）を検出し、必要なファイルだけを効率的にコピー（アップロード/ダウンロード）し、同期させるコマンドです。

https://docs.aws.amazon.com/cli/v1/reference/s3/sync.html

また、`.github/workflows`で、`aws s3 sync`を使用するときのオプションとして、`--exclude`というものが存在し、下記の記事に詳細が記載してあるので、今回使用してみました。

https://github.com/marketplace/actions/s3-sync

> **オプションのヒント:** リポジトリのルートディレクトリ全体をアップロードする場合は、`--exclude '.git/*'` を追加すると、`.git` フォルダが同期されるのを防げます。`.git` フォルダにはソースコードの履歴が含まれているため、クローズドソースのプロジェクトでは同期されると履歴が公開されてしまう可能性があります。
>
> （複数のパターンを除外したい場合は、除外するパターンごとに `--exclude` オプションを指定する必要があります。また、シングルクォート `'` を付けることも重要です。）
>

こちらのオプションは、上記より、ワークフロー内で、`--exclude '.git/*'`とすることで、index.htmlやcss/ をS3へアップロードし、.git フォルダが同期されるのを防ぎます。



今回は、必要なファイルだけを S3 に同期するため、`--exclude "*"` で全除外してから対象ファイルを `--include` で指定しています。
これは、`.git` や `.terraform` ディレクトリなどを誤ってアップロードしないための安全策になります。


**③`--delete` オプション**
また、ワークフロー内で、`aws s3 sync`を使用するときのオプションとして、`--delete`がございます。

https://github.com/marketplace/actions/s3-sync

上記記事に記載されている詳細を日本語訳すると下記になります。

>最も重要なのは、--delete オプションを付けると、最新のリポジトリ（またはビルド結果）に存在しないファイルは、S3 バケットから完全に削除されるということです。

つまり、S3バケットの中身を、ローカル（デプロイ元）の内容と同じ状態に保つようにするためのオプションになります。

なので、今回 `--delete`を使用し、S3 側に存在するがローカルには存在しないファイルを削除するようにしました。
このようにすることで、古いファイルが残り続けて想定外の挙動を引き起こすのを防ぐことができます。



# GitHub Secrets の設定

以下の 4 つを GitHub の Settings > Secrets and variables > Actions に登録します。

| シークレット名 | 値 |
|---|---|
| `AWS_ROLE_ARN` | Terraform の output で出力された IAM ロール ARN |
| `AWS_REGION` | `ap-northeast-1` |
| `S3_BUCKET_NAME` | デプロイ先の S3 バケット名 |
| `CLOUDFRONT_DISTRIBUTION_ID` | CloudFront ディストリビューション ID |

設定方法は下記を参考にしていただければ幸いです。

https://docs.github.com/ja/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets

## デプロイフロー全体像

今回作成したワークフローのデプロイフロー全体像をまとめると下記になります。

```
1. PR を作成 / 更新 (opened / synchronize / reopened)
       │
2. GitHub Actions トリガー
       │
3. GitHub OIDC エンドポイントからトークン取得
       │
4. AWS STS: AssumeRoleWithWebIdentity
   ├─ OIDC プロバイダーの検証
   ├─ aud 条件チェック (sts.amazonaws.com)
   └─ sub 条件チェック (repo:<owner>/<repo>:*)
       │
5. 一時クレデンシャル発行（有効期限付き）
       │
6. aws s3 sync → S3 バケットに index.html, images/* を同期
       │
7. aws cloudfront create-invalidation → CDN キャッシュをクリア
       │
8. ユーザーが CloudFront 経由で最新コンテンツを取得
```


# まとめ

今回の実装で実現できたことは、以下のとおりです。

- **アクセスキー不要**：OIDC により一時クレデンシャルのみで認証。長期クレデンシャルの管理リスクがゼロ
- **最小権限**：S3 の特定バケットと CloudFront の特定ディストリビューションに対するデプロイ操作のみを許可
- **リポジトリ制限**：信頼ポリシーの `sub` 条件で、指定リポジトリからのアクセスのみに絞り込み
- **インフラのコード管理**：OIDC プロバイダー・IAM ロール・ポリシーを全て Terraform で管理し、再現性を確保

GitHub Actions と AWS OIDC の組み合わせはセキュリティ面でのメリットが大きく、新しいプロジェクトでも積極的に採用したい構成です。Terraform モジュール化しておけば、別プロジェクトへの流用も容易です。


# 参考

https://docs.github.com/ja/actions/concepts/security/openid-connect

https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_openid_connect_provider

https://github.com/Onepiece2424/bedrock-ai-portfolio
