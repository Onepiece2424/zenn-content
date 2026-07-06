---
title: "Terraformを用いたVPC+EC2を組み合わせたマルチAZネットワーク構成の作成"
emoji: "😄"
type: "tech"
topics:
  - "aws"
  - "terraform"
  - "vpc"
  - "ec2"
published: true
published_at: "2026-07-07 07:15"
---

## はじめに

マイクロサービスアプリのインフラをTerraformで構築する際、リソースをフラットに書き続けると可読性・再利用性が急速に落ちます。
今回はネットワーク層とコンピュート層をモジュールに分割し、複数環境（devやprodなど）への展開を見据えた構成に整理しました。その設計方針と工夫した点をまとめていきます。

インフラ構成は、VPC＋EC2といったかたちで、シンプルな構成にしていますので、Terraformを使い始めた方の参考になれば幸いです（今回作成したインフラ構成は下記になります）。


![](https://static.zenn.studio/user-upload/540a85223dbc-20260707.png)


## プロジェクト構成
今回作成したプロジェクトに関しては、下記のディレクトリ構成にしております。

```
.
├── environments/
│   └── dev/
│       ├── main.tf          # モジュール呼び出し
│       ├── variables.tf     # 環境変数の型定義
│       ├── dev.tfvars       # 環境ごとの実値
│       └── terraform.tf     # バックエンド・プロバイダ設定
└── modules/
    ├── network/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── compute/
        ├── main.tf
        └── variables.tf
```

ディレクトリ構造の方針は「**環境 × レイヤー**」で分けることです。
工夫したことは、moduleを使用した複数環境への展開を見据えたリソース構成にすることです。
ディレクトリ構成の設計段階から、`environments/` ディレクトリ以下で環境差分（dev/stg/prod）を吸収し、`modules/` ディレクトリ以下でインフラ層（network/compute）の実装を持つように設計いたしました。



## モジュール化の設計

### networkモジュール

networkモジュールは、作成したnetworkディレクトリのことで、VPCまわりをまるごと担当するモジュールになります。東京リージョンの2AZ（ap-northeast-1a / 1c）を前提に、パブリック・プライベートそれぞれ2サブネットを作成いたしました。

```hcl:modules/network/main.tf

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = var.env
  }
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_subnet_1a.id
  depends_on    = [aws_internet_gateway.igw]
}
```

工夫したことは、NAT Gatewayに`depends_on`でIGW（インターネットゲートウェイ）への依存を明示したところです。
`depends_on`は、Terraformに「このリソースは別のリソースが作成された後に作成してください」と明示的に伝えるための設定になります。
Terraformは依存関係を自動解析するが、暗黙的な依存（参照ではなくリソースの存在が前提になるケース）は明示的に書いておくほうが安全になります。

https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway

また、モジュールの外に出す情報は `outputs.tf` で絞りました。

```hcl:modules/network/outputs.tf

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = [
    aws_subnet.public_subnet_1a.id,
    aws_subnet.public_subnet_1c.id
  ]
}

output "private_subnet_ids" {
  value = [
    aws_subnet.private_subnet_1a.id,
    aws_subnet.private_subnet_1c.id
  ]
}
```

外部に公開するのは「IDだけ」に留め、CIDRや詳細設定はモジュール内部で完結させ、呼び出し側がネットワークの実装詳細を知らなくて済む構造にしました。

---

### computeモジュール

computeモジュールは、computeディレクトリ配下のリソース群で、EC2インスタンス（踏み台サーバ・Webサーバ）とセキュリティグループを管理するモジュールになります。networkモジュールの出力をそのまま受け取るようにしました。

```hcl:modules/compute/main.tf

data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
  owners = ["099720109477"]
}

resource "aws_instance" "bastion_server" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  subnet_id                   = var.public_subnet_ids[0]
  associate_public_ip_address = true
  vpc_security_group_ids      = [aws_security_group.public_sg.id]
  key_name                    = aws_key_pair.ssh_key.key_name
}

resource "aws_instance" "web_server" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  subnet_id                   = var.private_subnet_ids[0]
  associate_public_ip_address = false
  vpc_security_group_ids      = [aws_security_group.private_sg.id]
  key_name                    = aws_key_pair.ssh_key.key_name
}
```

工夫したところは、AMIのIDをハードコードせず `data "aws_ami"` で動的に取得したところです。
AMIはリージョンごと・時期ごとにIDが変わるため、フィルタで最新のUbuntu 22.04 LTSを毎回解決する設計にしました。
これにより、将来的なAMI更新時も`main.tf`を変更する必要がないようにしました。

https://developer.hashicorp.com/terraform/language/data-sources
https://developer.hashicorp.com/terraform/language/block/data

---

### セキュリティグループの構成

今回、SG（セキュリティグループ）に関しては、踏み台→プライベートへのSSH許可を「IPアドレス指定」ではなく「**セキュリティグループ参照**」で実装しました。

```hcl:modules/compute/main.tf
# プライベートSGのインバウンドルール
resource "aws_vpc_security_group_ingress_rule" "allow_tls_ipv4_private_sg" {
  security_group_id            = aws_security_group.private_sg.id
  referenced_security_group_id = aws_security_group.public_sg.id  # IPではなくSGを参照
  from_port                    = 22
  ip_protocol                  = "tcp"
  to_port                      = 22
}
```

ここでは、踏み台のIPアドレスは動的に変わりうるため、IP指定ではなく「踏み台に紐づいたSGからの通信を許可」という形にするため、`referenced_security_group_id`を用いて、通信許可するSGを設定しました。
なので、踏み台のIPが変わってもプライベート側のルールを更新する必要がなく、管理コストが下がるようになりました。

https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc_security_group_egress_rule#referenced_security_group_id-1


## 環境分離の設計

### tfvarsで環境差分を管理

```hcl:environments/dev/dev.tfvars
my_ip    = "xx.xx.xx.xx"
env      = "dev"
vpc_cidr = "10.0.0.0/16"
```

今回、git管理下からは除外しておりますが、`dev.tfvars`も作成し、環境ごとに変わる値（VPC CIDR、環境名）だけを `*.tfvars` に切り出すようにしました。
このようにすることで、将来 `stg` 環境を追加する場合は `environments/stg/` を作り、`stg.tfvars` に別の値を書くだけで同じモジュールを使い回せるようになります。

https://dexall.co.jp/articles/?p=2337

### リモートバックエンド（S3）

```hcl:environments/dev/terraform.tf

backend "s3" {
  bucket = "task-microservices-app-remote-backend-bucket"
  key    = "dev/terraform.tfstate"
  region = "ap-northeast-1"
}
```

今回、リモートバックエンドとして、S3を作成し、tfstateをS3で管理することでチーム開発でのstateの競合を防ぐようにしました。
また、keyのパスを `dev/terraform.tfstate` のように環境名をプレフィックスにすることで、同一バケット内で複数環境（devやprodなど）のstateを分離するようにもしました。
このようにすることで、開発環境とは別の環境を作成し、`prod/terraform.tfstate` のようにすることで、別の環境のstateファイルを管理できるようにしました。

https://developer.hashicorp.com/terraform/language/backend/s3

### モジュール間の依存はoutputで繋ぐ

```hcl:environments/dev/main.tf

module "vpc" {
  source   = "../../modules/network"
  env      = var.env
  vpc_cidr = var.vpc_cidr
}

module "ec2" {
  source             = "../../modules/compute"
  env                = var.env
  my_ip              = var.my_ip
  vpc_id             = module.vpc.vpc_id            # networkモジュールのoutputを注入
  public_subnet_ids  = module.vpc.public_subnet_ids
  private_subnet_ids = module.vpc.private_subnet_ids
}
```

ここでは、computeモジュールはnetworkモジュールの出力を受け取るだけで、VPCやサブネットの実装には依存しないようにしました。
インターフェース（variables/outputs）さえ維持すれば、モジュール内部を差し替えても呼び出し側に影響が出ないかと思います。



## まとめ

今回の設計で意識したポイントを整理すると、下記になります。

| 工夫した点 | 効果 |
|---|---|
| networkとcomputeをモジュール分割 | 責務が明確になり、レイヤーごとに独立して変更できる |
| outputでIDのみを公開 | モジュール間の結合度を最小化 |
| `data "aws_ami"` で動的AMI取得 | AMI更新をコードレスで追従 |
| SGのSGソース参照 | IPアドレス管理の手間をなくす |
| tfvarsで環境差分を吸収 | 同じモジュールをdev/stg/prodで使い回せる |
| S3バックエンド + keyパス分離 | 複数環境のstateを安全に管理 |

一番重要なのは、*環境をまたいで再利用できる構成を最初から意識しておくこと* で、`environments/stg` を追加するコストが大幅に下がるようにすることです。
インフラも「コードとして育てていく」設計が大事だと実感しました。



## 動作確認：SSH接続からインターネット疎通まで

インフラを `terraform apply` した後、実際にEC2へSSH接続できるか・プライベートサブネットからインターネットに出られるかを手順を追って確認できるようにもしております。
手順としては、下記の通りです。

### 1. キーペアの作成と配置

今回、TerraformのcomputeモジュールはSSH公開鍵を `~/.ssh/ec2-keypair-new.pub` から読み込む設定にしました。

```hcl:modules/compute/main.tf
resource "aws_key_pair" "ssh_key" {
  key_name   = "ssh_key"
  public_key = file(pathexpand("~/.ssh/ec2-keypair-new.pub"))
}
```

そのため、ローカルPC上で、 `terraform apply` の前にキーペアを用意しておく必要があります。

```bash
# ed25519でキーペアを生成（-f でファイル名を指定）
ssh-keygen -t ed25519 -f ~/.ssh/ec2-keypair-new
```

*Ed25519* とは、非常に高速かつ安全で、鍵サイズが小さい次世代の楕円曲線暗号（公開鍵暗号）の一種です。従来のRSA暗号などに比べて短い鍵長で同等以上のセキュリティ強度を保てるため、GitHubへの接続やサーバー管理（SSH接続）において現在最も推奨されている方式です。

上記を実行すると `~/.ssh/ec2-keypair-new`（秘密鍵）と `~/.ssh/ec2-keypair-new.pub`（公開鍵）の2ファイルが作成されます。

すでに別の場所にキーペアを生成してしまった場合は、`mv`コマンドなどを使用し、 `~/.ssh/` に移動するようにしてください。

```bash
mv ~/ec2-keypair     ~/.ssh/
mv ~/ec2-keypair.pub ~/.ssh/
```


### 2. ssh-agentに秘密鍵を登録

*ssh-agent* は、コンピューター間の安全な通信（SSH接続）で使う「秘密鍵」を記憶し、パスワード入力の手間を省くプログラムです。
通常、秘密鍵にはセキュリティのために「パスフレーズ」が設定されていますが、これを一度入力するだけで、以降は自動的に認証を行ってくれます。

今回、踏み台からプライベートEC2へ接続する際、**エージェント転送（-A オプション）** を使うようにしました。
これにより手元の秘密鍵をプライベートEC2に置かずに多段SSH接続が可能になります。

まず、手元のssh-agentに秘密鍵を登録します。

```bash
ssh-add ~/.ssh/ec2-keypair-new
```

登録できているか確認するには `ssh-add -l` コマンドを実行し、確認してください。



### 3. 踏み台サーバへSSH接続

```bash
ssh -A ubuntu@<踏み台のパブリックIP>
```

上記の `-A` はエージェント転送を有効にするオプションになります。
これを付けておくと、踏み台に入った後でもローカルのssh-agentを経由して認証が行えるため、プライベートEC2にも秘密鍵なしで接続できます。

踏み台のパブリックIPはAWSコンソールの「EC2 > インスタンス > public-ec2」の「パブリック IPv4 アドレス」から確認してください。
また、 `terraform output` でoutputを定義していれば即座に取得できます。



### 4. プライベートEC2へSSH接続（踏み台経由）

踏み台に入ったまま、下記のコマンドを実行し、プライベートEC2へさらにSSH接続してください。

```bash
ssh ubuntu@<プライベートEC2のプライベートIP>
```

プライベートIPはAWSコンソールの「EC2 > インスタンス > private-ec2」の「プライベート IPv4 アドレス」から確認してください。

セキュリティグループの設定でプライベートSGのインバウンドは「踏み台のSGからのポート22のみ許可」としているため、インターネットから直接この接続を行うことはできません。

エージェント転送が正しく機能していれば、踏み台経由でパスワードなしに入ることができます。



### 5. インターネット疎通確認

プライベートサブネットのEC2はNAT Gatewayを経由してインターネットへ出られる設計になっております。SSH接続後、以下のコマンドで疎通を確認してください。

#### 方法1: ping（手軽な確認）

```bash
ping -c 4 google.com
```

DNS解決とICMPの疎通が同時に確認できます。NATが正しく機能していれば返答が返ってきます。

#### 方法2: curl（おすすめ）

```bash
curl https://www.google.com
```

HTTPSレベルでの疎通確認です。実際のアプリがHTTPS通信できるかの確認に近いです。

```bash
curl ifconfig.me
```

実行すると、グローバルIPが返ってきます。NAT Gatewayに付けたElastic IPが表示されれば、NATを経由して外に出ていることが確認できます。

#### 方法3: nslookup（DNS確認）

```bash
nslookup google.com
```

DNSが解決できているかを確認します。VPCのDNS設定に問題があると名前解決だけ失敗するケースがあるため、pingと組み合わせて確認すると原因の切り分けがしやすいです。

#### 方法4: apt update（Ubuntuらしい確認）

```bash
sudo apt update
```

パッケージリポジトリへの実際のHTTPS通信が発生するため、本番に近い形での疎通確認になります。`0 packages can be upgraded` のような出力が得られれば問題ないです。

#### 方法5: ルーティングの確認

```bash
ip route
```

デフォルトゲートウェイがVPCのルーターを向いているかを確認してください。NATの設定に問題がある場合、ここでルートが意図通りに設定されていないことが分かります。

#### 方法6: DNS設定の確認

```bash
cat /etc/resolv.conf
```

DNSサーバの設定を確認してください。AWSのVPC内ではデフォルトで `169.254.169.253` または VPC CIDRの +2 アドレスがDNSとして設定されます。名前解決が通らない場合の調査起点になります。

https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/AmazonDNS-concepts.html



### 疎通確認の流れまとめ

```
ローカル
  └─ ssh -A ubuntu@<踏み台パブリックIP>   ← セキュリティグループ: 自分のIPからSSH許可
       └─ ssh ubuntu@<プライベートIP>      ← セキュリティグループ: 踏み台SGからSSH許可
            └─ curl ifconfig.me           ← NAT Gateway経由でインターネットへ
```

NAT Gatewayが正しく機能していれば `curl ifconfig.me` の返り値はNAT GatewayのElastic IPになります。
これが確認できれば、プライベートサブネットからの外向き通信が意図通りにルーティングされていることの証明になるかと思います。

## 参考
今回紹介したコードに関しては、下記のリポジトリに保存しておりますので、よかったら参考にしていただけますと幸いです。

https://github.com/Onepiece2424/task-microservices-app
