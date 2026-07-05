---
title: "API Gateway のステージ変数をわかりやすく解説"
emoji: "🐕"
type: "tech"
topics:
  - "aws"
  - "apigateway"
published: true
published_at: "2025-09-15 15:46"
---

# はじめに
AWS API Gateway には「ステージ変数」という便利な仕組みがあります。これは簡単にいうと ステージごとに持てるキーと値のペア で、環境変数のように使うことができます。

例えば、同じ API を 開発環境（dev）・テスト環境（beta）・本番環境（prod） で使いたい場合、それぞれの環境ごとに接続先を変えたいことがありますよね。そんなときに役立つのがステージ変数です。

# ステージ変数 とは？
ステージ変数を簡単にまとめると、下記のような特徴があります。

- API Gateway の デプロイステージ（例: dev, beta, prod）に紐づけて設定できるキーと値
- API の統合リクエストやマッピングテンプレート内で利用可能
- 環境変数のように動作し、ステージごとに異なる設定を持てる

ポイントは 同じ API 定義をステージごとに挙動を変えられる ということです。

ただし、注意点として、ステージ変数は 認証情報や秘密データの保存には使えません。
なので、機密データを統合に渡したい場合は、Lambda オーソライザーを使うのが推奨です。

# 使い方
**1. バックエンドの切り替え**
API を HTTP プロキシとして使うとき、環境ごとに異なるバックエンドに切り替え可能です。

```json
https://${stageVariables.url}/api
```

- dev ステージ → url=beta.example.com
- prod ステージ → url=example.com

ステージごとに呼び出す先を変えられるので、API 定義を一元管理できます。

**2. Lambda 関数の切り替え**
Lambda をバックエンドに使う場合、ステージごとに異なるエイリアスを指定できます。

```json
arn:aws:lambda:ap-northeast-1:123456789012:function:MyFunction:${stageVariables.lambdaAlias}
```

- dev ステージ → lambdaAlias=dev
- prod ステージ → lambdaAlias=prod

この場合は、Lambda 側で API Gateway に呼び出し権限を付与する必要があります。
AWS CLI では以下のように設定できます。

```json
aws lambda add-permission \
  --function-name "arn:aws:lambda:ap-northeast-1:123456789012:function:my-function" \
  --source-arn "arn:aws:execute-api:ap-northeast-1:123456789012:api_id/*/GET/resource" \
  --principal apigateway.amazonaws.com \
  --statement-id apigateway-access \
  --action lambda:InvokeFunction
```

**3. マッピングテンプレートでの利用**
ステージ変数はマッピングテンプレートの中でも使えます。
例えば、同じ Lambda 関数を使いながらも、ステージごとに異なる DynamoDB テーブルを参照したい場合です。

```json
{
  "TableName": "$stageVariables.tableName",
  "Key": {
    "id": "$input.params('id')"
  }
}
```

- dev ステージ → tableName=DevTable
- prod ステージ → tableName=ProdTable

これで 1 つの Lambda 関数でも環境差分を吸収できます。

# まとめ
まとめると、API Gateway のステージ変数には、以下の特徴があります。

- ステージごとに設定できるキーと値のペア
- 環境変数のように使える
- バックエンド URL の切り替え、Lambda エイリアスの切り替え、マッピングテンプレートでの利用が可能

ただし 認証情報には使わないことといった注意点があります。

複数の環境を運用するときに「同じ API を保ちつつ環境ごとに設定を切り替えたい」と思ったら、ぜひステージ変数を活用してみてください。

# 参考
https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/stage-variables.html#:~:text=%E3%82%B9%E3%83%86%E3%83%BC%E3%82%B8%E5%A4%89%E6%95%B0%E3%81%AF%E3%80%81REST%20API,%E3%82%92%E5%8F%82%E7%85%A7%E3%81%97%E3%81%A6%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82