# 第3章: ECRの構築とTerraformの基本操作

この章では、AWS のコンテナレジストリである ECR を Terraform を使って作成します。<br>

## 3.1 本章完了時点のサンプルコード

本章完了時点のサンプルコードは以下になります。必要に応じて参考にしてください。<br>

+ https://github.com/shonansurvivors/laravel-fargate-infra/tree/chapter-3 <br>

## 3.2 Terraform 用の GitHub リポジトリの作成(P26〜)<br>

Laravel のコードを管理するための GitHub リポジトリとして、laravel-fargate-app を作成しましたが、Terraform のコードは別リポジトリで作成することにします。<br>
本書では、laravel-fargate-infra リポジトリを作成し、そこでコード管理する前提で説明 を続けます。<br>
あなたの GitHub アカウントに laravel-fargate-infra リポジトリを作成しておいてください。<br>

## 3.3 .gitignoreの作成<br>

+ `$ touch laravel-fargate-infra/.gitignore`を実行<br>

+ `laravel-fargate-infra/.gitignore`を編集<br>

```:.gitignore
**/.terraform/*
*.tfstate
*.tfstate.*
*.tfvars
```
## 3.4 環境別のディレクトリの作成

+ `$ mkdir laravel-fargate-infra/envs && mkdir $_/{dev,stg,prod}`を実行<br>

+ `$ touch laravel-fargate-infra/envs/prod/.terraform-version`を実行<br>

+ `laravel-fargate-infra/envs/prod/.terraform-version`を編集<br>

```:.terraform-version
1.3.4
```

これにより、prod ディレクトリか、それより配下のディレクトリで tfenv install を実行すると、使用される Terraform のバージョンが 1.3.4 になります。<br>

+ laravel-fargate-infra/envs/prod`ディレクトリで `$ tfenv install`を実行<br>

## 3.6 目的別のディレクトリの作成

さらに、prod ディレクトリの直下に app ディレクトリ、さらにその直下に foobar ディレクトリを作成してください。<br>

+ `$ mkdir laravel-fargate-infra/envs/prod/app && mkdir $_/foobar`を実行<br>

app ディレクトリ配下には、アプリケーションに直接関係する AWS リソースの Terraform のコードを格納することにします。<br>
加えて、ひとつの環境の中で複数のアプリケーションの tfstate を独立して管理できるよう、アプリケーション別にディレクトリを作成することにしました。<br>

+ `i` __appディレクトリについて__<br>
  本書では最終的に ECR、ECS クラスター、ECS 関連の IAM ロールをここで管理します。<br>

+ `i` __foobar について__<br>
  foobar は日本語で言うと「ほにゃらら」のようなものです。実際にはアプリケーションを特定する固有名詞が入ることを想定しています。<br>

もし、上記のようにディレクトリを分けず、prod 直下に Terraform のコードを書くと、prod 環境のありとあらゆる AWS リソースがひとつの tfstate で管理されることになります。<br>
これによるデメリットとしては、以下が考えられます。<br>

+ Terraform のコードを実際の AWS リソースに適用 (apply) する際、もしミスがあった場合に、その影響が環境全体に波及するリスクが高い<br>

+ 管理する AWS リソースが増えてくると、ごく一部のリソースへの変更だとしても、Terraform の処理 (plan や apply) に時間がかかる<br>

そこで、本書では目的別のディレクトリを作成します。<br>
なお、本書でのディレクトリの分け方は、比較的細かい方だと思います。実務においては、もっと大まかなディレクトリ分割にしているケースもあるでしょう。<br>
Terraform でのディレクトリの分け方は、これといった決定的なベストプラクティスは無いので、本書の方法はあくまでひとつのやり方として参考にしてください。<br>

## 3.7 backendの設定(P29〜)

tfstate をどこに保存するのか、その設定を行います。<br>
設定は拡張子 tf のファイルであれば、どこに書いても良いのですが、本書では backend.tf という名前のファイルを作り、そこに記述するようにします。<br>
以下の通り backend.tf を作成してください。<br>

<img src="https://i.gyazo.com/6b691970ef305eacc3efd281130b1bab.png" /> <br>

+ `$ touch laravel-fargate-infra/envs/prod/app/foobar/backend.tf`を実行<br>

+ `envs/prod/app/foobar/backend.tf`を編集<br>

```tf:backend.tf
terraform {
  backend "s3" {
    bucket  = "takapepe-tfstate"
    profile = "terraform"
    key     = "example/prod/app/foobar_v1.3.4.tfstate"
    region  = "ap-northeast-1"
  }
}
```

__bucket__<br>

  前章で作成した、tfstate 保存用の S3 バケット名を指定してください。<br>

__key__<br>

  保存先の tfstate のファイル名を指定します。本書では、以下のネーミングルールとします。<br>

  + システム名/laravel-fargate-infra リポジトリにおける envs 以下のパス _Terraform バージョン名.tfstate <br>

  ここでのシステム名とは、複数のサービスを束ねる大きな括りだと思ってください。本書では、システム名を example とします。<br>

__region__<br>

  前章で作成した、tfstate 保存用の S3 バケットのリージョンを指定します。<br>

## 3.8 terraform fmt(P30〜)

先ほどの backend.tf では、「=」の位置が縦に揃っていますが、最初からこのように書く必要はありません。tf ファイルが存在するディレクトリで、terraform fmt と実行すると、自動で整形してくれます。<br>

+ `$ terraform fmt`を実行<br>

配下のディレクトリにある tf ファイルも含めて整形したい場合は、--recursive オプションを付けます。<br>

+ `$ terraform fmt --recursive`<br>

## 3.9 プロバイダーの設定(P30〜)

Terraform は AWS だけでなく GCP や Azure などにも対応しています。Terraform には各サービスに対応したプロバイダーと呼ばれるプラグインが存在し、これを使うことで各 サービスと連携します。<br>
本書では、AWS を取り扱うため、AWS のプロバイダーを使用することを宣言します。<br>
prod ディレクトリ配下に provider.tf を作成してください (foobar ディレクトリ配下では無いので注意してください)。<br>

+ `$ touch laravel-fargate-infra/envs/prod/provider.tf`を実行<br>

+ `envs/prod/provider.tf`を編集<br>

```tf:provider.tf
provider "aws" {
  region = "ap-northeast-1"
}

terraform {
  required_providers {
    aws = {
      source   = "hashicorp/aws"
      viersion = "3.42.0"
    }
  }
}
```

以上により、プロバイダーとして AWS が使用されます。<br>
また、プロバイダーにもバージョンがあり、ここでは使用するバージョンを 3.42.0 に固定しています。<br>
プロバイダーバージョンを固定することで、バージョンの差異による想定外のエラーなどを防ぐようにしています。<br>
なお、古い Terraform のバージョン (0.13 以前) では、プロバイダーのバージョンは以下のように provider ブロックに書くことになっていました。<br>

```tf:provider.tf
provider "aws" {
  region  = "ap-northeast-1"
  version = "3.42.0"
}
```

最新の Terraform では、この書き方は非推奨となっています。<br>

## 3.10 terraform バージョンの固定(P32〜)

併せて、Terraform のバージョンを固定します。以下の通り、required_version を追加してください (太字が追加箇所です)。<br>

+ `envs/prod/provider.tf`を編集<br>

```tf:provider.tf
provider "aws" {
  region = "ap-northeast-1"
}

terraform {
  required_providers {
    aws = {
      source   = "hashicorp/aws"
      viersion = "3.42.0"
    }
  }

  required_version = "1.3.4" // 追加
}
```

## 3.11 デフォルトのタグ設定(P32〜)

AWS の各リソースには基本的にタグを付けることができますが、Terraform の AWS プロバイダー 3.38 以降から全てのリソースに一括して共通のタグを付けられるようになりました。<br>
以下の通り、default_tags を追加してください (太字が追加箇所です)。<br>

+ `envs/prod/provider.tf`を編集<br>

```tf:provider.tf
provider "aws" {
  region = "ap-northeast-1"

  default_tags {
    tags = {
      Env    = "prod"
      System = "example"
    }
  }
}

terraform {
  required_providers {
    aws = {
      source   = "hashicorp/aws"
      viersion = "3.42.0"
    }
  }

  required_version = "1.3.4"
}
```

本書では、System タグにシステム名を、Env タグに環境名を付けることにします。<br>

## 3.12 シンボリックリンクの作成(P33〜)

envs/prod ディレクトリに provider.tf を作成しましたが、envs/prod/foobar ディレクトリにあるわけではないので、foobar ディレクトリでの Terraform の実行時に読み込んではくれません。<br>
そこで、foobar ディレクトリにはシンボリックリンクを作成し、そこから prod ディレクトリの provider.tf を参照するようにします。<br>
foobar ディレクトリで以下コマンドを実行してください。<br>

+ `$ ln -fs ../../provider.tf provider.tf`を実行<br>

<img src="https://i.gyazo.com/cec50844088c5c7686c3c4d0cdfe5140.png" /> <br>

もし各ディレクトリで個別に provider.tf の実体を作ってしまうと、その内容を修正する場合に、各ディレクトリのファイルを修正して回らなければならなくなります。<br>
シンボリックリンクを利用することで、ひとつの provider.tf で定義した内容を各ディレクトリで共通して使用できるようにしています。<br>

## 3.13 terraform init の実行(P33〜)<br>

ここまでのファイル作成が完了したら、foobar ディレクトリで、terraform init を実行してください。<br>

+ `$ terraform init`を実行<br>

以下のようなメッセージが表示されれば問題ありません。<br>

```:terminal
Terraform has been successfully initialized!
```

今回のように backend やプロバイダーを設定したり、今後モジュールと呼ばれるものを追加した場合は terraform init を実行するようにしてください。<br>
なお、Terraform v0.14 以降からは terraform init 実行後に、以下のように.ter-raform.lock.hcl が作成されますが、これは Git 管理して構わないファイルです<br>

<img src="https://i.gyazo.com/8dbb4e05ba0989f8e49361e4079a5da4.png" /> <br>

## 3.1.4 ECR用の tf ファイルの作成（P34〜)

ここからは、AWS のリソースを作成していきます。<br>
AWS Fargate で Docker のコンテナを動かすためには、事前にその Docker イメージを コンテナレジストリに登録しておく必要があります。<br>
AWS には、ECR(Elastic Container Registry) というコンテナレジストリのサービスが提供されているので、本書ではこれを利用します。<br>
まず、以下の通り ecr.tf を作成してください。ここに ECR の内容を記述していきます。<br>

<img src="https://i.gyazo.com/1a4e74bf1f4b825fb43d74b249cb0350.png" /> <br>

+ `$ touch laravel-fargate-infra/envs/prod/app/foobar/ecr.tf`を実行<br>

tf ファイルの名前に関しては特に決まりはありませんが、以下のいずれかのケースが取ら れることが多いかと思います。<br>

+ main.tf を作成し、そこに様々な種類のリソースを記述する<br>

+ iam.tf など、ある程度のリソースの種類ごとに tf ファイルを作成し、それぞれに対応するリソースを記述する<br>

本書では後者の方法を採用します。<br>

## 3.14.1 aws_ecr_repository の作成(P35〜)

ecr.tf を以下の通り編集してください。<br>

```tf:ecr.tf
resource "aws_ecr_repository" "nginx" {
  name = "example-prod-foobar-nginx"

  tags = {
    Name = "example-prod-foobar-nginx"
  }
}
```

ローカルの docker 環境では、以下の 3 コンテナが動いていました。<br>

1. web コンテナ : nginx が動くコンテナ<br>

2. app コンテナ : php が動くコンテナ<br>

3. db コンテナ : MySQL が動くコンテナ<br>

このうちの nginx のイメージを登録するリポジトリを今回作成しようとしています (な お、MySQL は Amazon RDS で動かします)。<br>

## 3.14.2 resource ブロック

resource ブロックに記述されている意味は以下になります。<br>

```tf:sample.tf
resource "リソースの種類" "リソースに付ける名前" {
  // このリソースの設定に使用する様々な引数
}
```

__リソースの種類__<br>

今回、ECR を作成するため、リソースの種類は aws_ecr_repository を指定します。<br>
なお、各リソース種類ごとに引数をどのように指定すれば良いかは、Terraform Registry というサイトの AWS に関するページ*1に記載されています。<br>

+ `＊3` https://registry.terraform.io/providers/hashicorp/aws/latest/docs <br>

__リソースに付ける名前__<br>

さらにリソースに名前を付けます。これは Terraform で管理する上での名前なので、AWS のリソースにその名前がつくわけではありません。<br>

__name__

引数 name は ECR のリポジトリ名となります。AWS のマネジメントコンソール上などにも表示される名前となります。<br>

__tags__

引数 tags で、タグ情報を付けます。本書では、各リソースに対して基本的に Name タグ を付けるようにしていきます。<br>

## 3.15 本書でリソース命名規則（P36〜)

本書でのリソース命名規則について解説します。<br>

### 3.15.1 Terraform リソースとしての名前

Terraform 公式サイトではありませんが、terraform-best-practices.com*2によれば、名前の付け方のベストプラクティスは以下になります。<br>
本書もこれに沿って、名前を付けるようにします。<br>

+ `＊3` https://www.terraform-best-practices.com/naming <br>

+ アンダースコア区切りにする (スネークケース)<br>
+ 小文字と数字のみを使う<br>
+ リソースの種類を名前に含めない<br>
+ その種類のリソースを 1 つだけ作成するような場合は、this と付ける<br>

## 3.15.2 AWS リソースとしての名前

本書では、各 AWS リソースの名前は以下のルールに沿って付けるようにしていきます。<br>

+ ハイフン区切りとする (ケバブケース)<br>
+ ecr や repository のような、リソースの種類を示すような名前は含めない<br>
+ {システム名}-{環境名}-{サービス名}に相当するプレフィックスを付ける<br>

## 3.16 terraform plan / apply の実行

まだ nginx の aws_ecr_repository しか記述できていませんが、ここで実際に AWS にリ
ソースを作成してみましょう。<br>

### 3.16.1 terraform plan

まず、terraform plan を実行してください。terraform plan により、どのような AWS リソースが作成されるかを事前確認できます。<br>

+ `laravel-fargate-infra/envs/prod/foobar / $ terraform plan`を実行<br>

```:terminal
$ terraform plan

Terraform used the selected providers to generate the following
execution plan. Resource actions are indicated with the following
symbols:
  + create

Terraform will perform the following actions:

  # aws_ecr_repository.nginx will be created
  + resource "aws_ecr_repository" "nginx" {
      + arn                  = (known after apply)
      + id                   = (known after apply)
      + image_tag_mutability = "MUTABLE"
      + name                 = "example-prod-foobar-nginx"
      + registry_id          = (known after apply)
      + repository_url       = (known after apply)
      + tags                 = {
          + "Name" = "example-prod-foobar-nginx"
        }
      + tags_all             = {
          + "Env"    = "prod"
          + "Name"   = "example-prod-foobar-nginx"
          + "System" = "example"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

ecr.tf に記述したリソースが作成されることがわかります。<br>
aws_ecr_repository の nginx の引数には name と tags しか記述しませんでしたが、それら以外にも様々な属性が設定されることがわかります。<br>

__image_tag_mutability__ <br>

image_tag_mutability は、Docker イメージとしてのタグの変更が可能かどうかの設定です。<br>
ここではデフォルトの MUTABLE(変更可能) となっています。<br>

__tags_all__<br>

tags_all は Terraform 独自の属性で、AWS リソースそのものにこうした属性はありません。<br>
tags_all の部分には provider.tf の provider ブロックで指定したデフォルトのタグと、リソース個別に指定したタグ (tags の内容) が設定されます。<br>
最終的に、AWS リソースとしての ECR のタグには Env, Name, System が設定されます。<br>

__known after apply__ <br>

known after apply と表示されている箇所は、実際に AWS リソースが作成されて初めて値が決まる属性です。<br>

__add / change / destroy__ <br>

最終行には「1 to add」と表示されていますが、リソースが 1 つ追加されることを意味しています。<br>
続いて「0 to change, 0 to destroy」と表示されていますが、リソースの変更や削除は無いということを意味しています。<br>
plan 後はまず最初にこの最終行を見て意図した通りにリソースが作成、変更、削除されようとしているか確認し、その次にリソース個別の変更内容を確認すると良いと思います。<br>

## 3.16.2 terraform apply の実行

+ `laravel-fargate-infra/envs/prod/foobar / $ terraform apply`を実行<br>

```:terminal
$ terraform apply

Terraform used the selected providers to generate the following
execution plan. Resource actions are indicated with the following
symbols:
  + create

Terraform will perform the following actions:

  # aws_ecr_repository.nginx will be created
  + resource "aws_ecr_repository" "nginx" {
      + arn                  = (known after apply)
      + id                   = (known after apply)
      + image_tag_mutability = "MUTABLE"
      + name                 = "example-prod-foobar-nginx"
      + registry_id          = (known after apply)
      + repository_url       = (known after apply)
      + tags                 = {
          + "Name" = "example-prod-foobar-nginx"
        }
      + tags_all             = {
          + "Env"    = "prod"
          + "Name"   = "example-prod-foobar-nginx"
          + "System" = "example"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

terraform apply を実行すると、まず最初は terraform plan を実行した時と同様に、これからどのようにリソースが作成、変更、削除されようとしているかが表示されます。<br>
ここで、yes と入力すると実際に AWS リソースが作成、変更、削除されます。<br>

```:terminal
Enter a value: yes

aws_ecr_repository.nginx: Creating...
aws_ecr_repository.nginx: Creation complete after 0s [id=example-prod-foobar-nginx]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

## 3.17 terraform state list / show / pull (P39〜)

ここから、いくつかの terraform の便利なコマンドを紹介します。<br>

## 3.17.1 terraform state list

+ `laravel-fargate-infra/envs/prod/foobar / $ terraform state list`を実行<br>

```:terminal
$ terraform state list

aws_ecr_repository.nginx
```

## 3.17.2 terraform state show

terraform state show は、指定したリソースについて、tfstate 上でどのような属性や値を 持っているか見ることができます。<br>
リソースの指定方法は、先ほどの terraform state list で表示された形式を使ってください。<br>

+ `laravel-fargate-infra/envs/prod/foobar / $ terraform state show aws_ecr_repository.nginx`を実行<br>

```:terminal
$ terraform state show aws_ecr_repository.nginx

# aws_ecr_repository.nginx:
resource "aws_ecr_repository" "nginx" {
    arn                  = "arn:aws:ecr:ap-northeast-1:236430135202:repository/example-prod-foobar-nginx"
    id                   = "example-prod-foobar-nginx"
    image_tag_mutability = "MUTABLE"
    name                 = "example-prod-foobar-nginx"
    registry_id          = "236430135202"
    repository_url       = "236430135202.dkr.ecr.ap-northeast-1.amazonaws.com/example-prod-foobar-nginx"
    tags                 = {
        "Name" = "example-prod-foobar-nginx"
    }
    tags_all             = {
        "Env"    = "prod"
        "Name"   = "example-prod-foobar-nginx"
        "System" = "example"
    }

    encryption_configuration {
        encryption_type = "AES256"
    }

    image_scanning_configuration {
        scan_on_push = false
    }
}
```
