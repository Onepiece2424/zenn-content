---
title: "ECS × Secrets Manager：ARN末尾の『::』はどこから来るのか"
emoji: "🐸"
type: "tech"
topics:
  - "aws"
  - "ecs"
  - "secret manager"
published: true
published_at: "2026-07-13 06:22"
---

# はじめに

ECSのタスク定義でSecrets Managerのシークレットを環境変数として渡そうとしたとき、こんなARNを見たことはないでしょうか？

```
arn:aws:secretsmanager:region:account_id:secret:secret_name-AbCdEf:username1::9d4cb84b-ad69-40c0-a0ab-cead3EXAMPLE
```

`username1` の後ろに `::` が並んでいます。一見するとタイポにも見えるし、コピペミスを疑いたくなる見た目をしています。しかし、これは正しい構文であり、ちゃんと意味があります。今回はこの `::` がどこから来るのか、なぜ省略できないのかを整理してみたいと思います。

## （重要）ARNの末尾は3パーツで構成されている

Secrets Managerのシークレットを参照するARNは、`secret-name` の後ろに最大3つの追加パラメータをコロン区切りで指定できます。

```
arn:aws:secretsmanager:region:account_id:secret:secret-name:json-key:version-stage:version-id
```

| パラメータ | 意味 | 省略時の挙動 |
|---|---|---|
| `json-key` | JSON形式のシークレットから特定のキーだけを取り出す | シークレットの内容全体を使う |
| `version-stage` | 参照するバージョンのステージングラベル | `AWSCURRENT`（最新）を使う |
| `version-id` | 参照するバージョンの一意のID | `AWSCURRENT`（最新）を使う |

3つとも省略可能なオプションパラメータです。ここまでは公式ドキュメントにも書いてある通り、特に難しい話ではないかと思います。問題は「一部だけ省略したいとき」に起きます。

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/secrets-envvar-secrets-manager.html

## なぜARNの中に「::」があるのか

ドキュメントでは、下記のように記載されています。

> 追加のパラメータはオプションですが、これらを使用しないでデフォルト値を使用する場合は、コロン : を含める必要があります。

つまり、追加のパラメータ `json-key`、`version-stage`、`version-id` の中から、どの組み合わせで使用する・しないかを決めることによって、Secrets ManagerのARNに「::」を含むようになります。

まとめると、 `::` は「タイポ」でも「謎の記号」でもなく、**「ここには何も入れません」という明示的なプレースホルダー** を意味を表します。

## パターン別に見る `:` の使用例

実際にどう書けばいいのか、ドキュメントを参考にパターンごとに整理してみます。

### シークレット全体を使う（最もシンプル）

```
arn:aws:secretsmanager:region:aws_account_id:secret:secret_name-AbCdEf
```

追加のパラメータ `json-key`、`version-stage`、`version-id` のどれも使用ない時の書き方。ARNの後ろに何も付けず、コロンも不要。

### JSONキーだけ指定する

```
arn:aws:secretsmanager:region:aws_account_id:secret:appauthexample-AbCdEf:username1::
```

上記のARNは、追加のパラメータ `json-key` を使用し、`version-stage`、`version-id` は使用しない時の書き方になります。`username1` は、特定のキーを表し、キー・バリューのキーとなります。

### バージョンステージだけ指定する

```
arn:...:secret:secret_name-AbCdEf::AWSPREVIOUS:
```

上記のARNは、追加のパラメータ `version-stage` を使用し、`json-key`、`version-id` は使用しない時の書き方になります。`AWSPREVIOUS` は、特定のバージョンのステージングラベルを表し、キー・バリューのキーとなります。


### バージョンIDだけ指定する

```
arn:aws:secretsmanager:region:aws_account_id:secret:appauthexample-AbCdEf:::9d4cb84b-ad69-40c0-a0ab-cead3EXAMPLE
```

上記のARNは、追加のパラメータ `version-id` を使用し、`json-key`、`version-stage` は使用しない時の書き方になります。`9d4cb84b-ad69-40c0-a0ab-cead3EXAMPLE` は、特定のバージョン IDを表し、キー・バリューのキーとなります。

### JSONキー＋バージョンステージ

```
arn:aws:secretsmanager:region:aws_account_id:secret:appauthexample-AbCdEf:username1:AWSPREVIOUS:
```

上記のARNは、追加のパラメータ `json-key`、`version-stage` を使用し、`version-id` は使用しない時の書き方になります。`username1`、`AWSPREVIOUS` は、特定のキーと特定のバージョンのステージングラベルを表し、キー・バリューのキーとなります。

### JSONキー＋バージョンID

```
arn:aws:secretsmanager:region:aws_account_id:secret:appauthexample-AbCdEf:username1::9d4cb84b-ad69-40c0-a0ab-cead3EXAMPLE
```

上記のARNは、追加のパラメータ `json-key`、`version-id` を使用し、`version-stage` は使用しない時の書き方になります。`username1`、`9d4cb84b-ad69-40c0-a0ab-cead3EXAMPLE` は、特定のキーと特定のバージョンIDを表し、キー・バリューのキーとなります。

## 覚え方のコツ

覚え方のコツとしては、Secret Manager のARN ＋ secrets オブジェクト ＋ 追加のパラメータ を表す下記の構文の表記順をざっくり把握しておくことかなと思います。

```
arn:aws:secretsmanager:region:aws_account_id:secret:secret-name:json-key:version-stage:version-id
```

## まとめ

今回は、ECSのタスク定義でSecrets Managerのシークレットを環境変数として渡そうとしたときの「::」について解説してみました。
今回の内容をまとめると、下記になります。

- ARN末尾の `json-key` / `version-stage` / `version-id` は**位置で意味が決まる**オプションパラメータ
- 途中のパラメータを省略する場合、コロンで位置を空けておかないと後続の値が誤って解釈される
- `::` は記号のバグではなく、「ここは意図的に空けています」という宣言

一見奇妙に見えるこの構文も、ARNが「名前付き引数」ではなく「位置引数」の集まりであることを理解すれば、驚くほどシンプルなルールだと分かるかと思います。

## 参考

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/secrets-envvar-secrets-manager.html
