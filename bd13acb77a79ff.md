---
title: "Cloud Storageに保存したSQLiteファイルからデータをリストア（復元）する方法"
emoji: "🔖"
type: "tech"
topics:
  - "docker"
  - "rails"
  - "cloudrun"
  - "gcs"
published: true
published_at: "2025-05-16 20:17"
---

## 背景

Dockerコンテナで動かしているRailsアプリケーションにおいて、データベースとしてSQLiteを使っている場合、「万が一コンテナが停止したらデータはどうなるのか？」という点が気になる場面がありました。
そこで、本記事では、Cloud Storageに保存していたSQLiteファイルを使って、別の環境（もしくは、ローカルマシン上で再起動したdockerコンテナ）でデータをリストアできるかを検証した手順を紹介します。

## 検証の目的

- コンテナが停止・破棄されたとしても、Cloud Storage上のSQLiteファイルを使えばデータをリストアできるのか？
- SQLiteファイルの参照先を変更することで、既存のデータを再利用できるのか？

## 検証環境

- Rails 8.0
- データベース　SQLite3
- コンテナ環境　Docker
- Google Cloud Storage（GCS）
- GCS バケットから取得した`production.sqlite3` ファイルを使用

今回使用したアプリケーションのアーキテクチャ図は、下記の通りです。

![](https://storage.googleapis.com/zenn-user-upload/4bef79cf0765-20250516.png)


## 検証手順

### ① Cloud Storage から SQLite ファイルを取得

まずは、Cloud Storage バケットに保存してある `production.sqlite3` をローカル環境へダウンロードしました。

今回は、docker 環境下のRails アプリケーションの `rails/storage/` ディレクトリ配下に、GCSから取得した`production.sqlite3` ファイルを保存しました。

### ② 既存データを削除

Railsアプリケーションを起動させたとき、SQLiteファイルを参照し、テーブルデータへインポートできるか検証するため、ローカルの開発環境で既に存在するデータベースを初期化しました。

なので、アプリケーション内の全てのデータの削除を行いました。

### ③ 一度コンテナを起動

この状態で一度 docker コンテナを立ち上げ、rails console を使用し、テーブルデータが何もない状態であることを確認します。

### ④ `database.yml` を編集し、Cloud Storage から持ってきた SQLite を参照させる

`config/database.yml` を編集し、SQLiteファイルの参照先を変更します。以下はその例です。

```yaml
default: &default
  adapter: sqlite3
  pool: 5
  timeout: 5000

development:
  <<: *default
  database: rails/storage/production.sqlite3　#GCSから取得したsqliteファイルのアプリケーション内の保存場所を参照させる

production:
  <<: *default
  database: /storage/production.sqlite3
```

上記だと、development 環境のDBでは、アプリケーション内の rails/storage ディレクトリに、GCS(GoogleCloudStorage)からダウンロードした production.sqlite3 ファイルを参照するということになります。
ポイントは、ローカルで配置したSQLiteファイルの**フルパス**を記述することです。このようにすることで、rails アプリケーションを再起動させたとき、GCS(GoogleCloudStorage)からダウンロードしたSQLiteファイルを参照し、テーブルへインポートするようになります。

### ⑤ 再度コンテナを起動し、データが復元されているか確認

最後にもう一度コンテナを立ち上げ、リストアされたSQLiteファイルが正しく読み込まれているかを確認しました。

実際にブラウザまたはRailsコンソールから確認すると、webサイト上に テーブルデータがリストア（復元）されていることを確認することができました。

## まとめ

docker 環境下の rails アプリで、Cloud Storage 上に保存していた SQLite ファイルをマウント・参照することで、**データをローカル環境で問題なくリストア（復元）できる**ことが確認できました。

これにより、以下のような場面で有効な手段になることが期待できます。

- コンテナの再構築時のデータのリストア
- 別環境でのデータ再利用（ステージング・検証環境など）
- 最小構成でのバックアップ運用

SQLiteは軽量な分、運用環境によっては敬遠されがちですが、小規模アプリやPoC開発においては、Cloud Storageと併用することで十分な柔軟性と安全性を担保できると感じました。

今回は Cloud Storage を使いましたが、AWS の S3 や他の外部ストレージに置き換えても同様の手法が応用可能なのではないかと考えております。

マウントや参照のパスさえ適切であれば、SQLiteの持つシンプルさを活かした柔軟な運用が可能です。

今回は、GCS(GoogleCloudStorage)から取得したSQLiteファイルをローカル環境でリストアする手法についてご紹介しました。

実際にやってみると「これなら最低限のバックアップ・リストア体制がとれる」と感じられる場面も多いと思います。

SQLite＋CloudStorageの組み合わせ、ぜひ一度試してみてください。