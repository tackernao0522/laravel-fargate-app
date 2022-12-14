## 3.17.3 terraform state pull (P41〜)

terraform state show では、ひとつひとつのリソース単位でしか情報を見ることができません。<br>
tfstate 内の複数のリソースの情報を見たい場合は、terraform state pull を使うと良いでしょう。tfstate ファイル全体が丸ごと出力されます。<br>
ある程度のリソース数になってくるとかなりの行数になるので、いったん以下のようにしてファイルに出力するのが良いかと思います。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform state pull > tmp.tfstate`を実行<br>

## 3.18 リソースの削除方法

ECR は作成しただけであれば料金はかかりませんが、今後その他の有料 AWS リソースを作成した時に備えて、リソースの削除方法を解説します。<br>

__3.18.1 削除したいリソースのコードを消した上で apply する__<br>

削除したいリソースのコードを消した上で apply すると、そのリソースは削除されます。<br>
例えば、以下のようにコメントアウトします。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar/ecr.tf`を編集<br>

```tf:ecr.tf
// resource "aws_ecr_repository" "nginx" {
//   name = "example-prod-foobar-nginx"

// tags = {
//    Name = "example-prod-foobar-nginx"
//  }
// }
```

その上で、terraform apply を実行して yes と入力すると、コード上で削除された (コメントアウトされた) リソースを AWS から削除できます。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform apply`を実行<br>

## 3.19 aws_ecr_lifecycle_policy の作成

ECR は保存されているイメージの容量に応じて料金がかかります。<br>
基本的には最新かそれに近いイメージをプルして Fargate で使用することになるので、もう使われることのない古いイメージが自動削除されるよう、[ライフサイクルポリシー](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/LifecyclePolicies.html) を設定します。<br>
ecr.tf に以下の通り追記してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar/ecr.tf`を編集(P43〜)<br>

```tf:ecr.tf
resource "aws_ecr_repository" "nginx" {
  name = "example-prod-foobar-nginx"

  tags = {
    Name = "example-prod-foobar-nginx"
  }
}

// 追加
resource "aws_ecr_lifecycle_policy" "nginx" {
  policy = jsonencode(
    {
      "rules" : [
        {
          "rulePriority" : 1,
          "description" : "Hold only 10 images, expire all others",
          "selection" : {
            "tagStatus" : "any",
            "countType" : "imageCountMoreThan",
            "countNumber" : 10
          },
          "action" : {
            "type" : "expire"
          }
        }
      ]
    }
  )

  repository = aws_ecr_repository.nginx.name
}
// ここまで
```

## 3.19.1 policy

policy には、どのようなルールでイメージを削除するかを記述します。<br>
ここでは、最新の 10 個までイメージを残し、それより古いものは自動削除されるように設定しています。<br>

## 3.19.2 json の記述について

policy のような、AWS において JSON 文字列での記述を要求される属性について、Terraform での記述方法は様々あります。<br>

1. ヒアドキュメントで記述する<br>
2. json 部分を別ファイルにして、[file 関数](https://www.terraform.io/docs/language/functions/file.html)や [templatefile 関数](https://www.terraform.io/docs/language/functions/templatefile.html)で呼び出す<br>
3. json で記述して [jsonencode 関数](https://www.terraform.io/docs/language/functions/jsonencode.html)で変換する<br>

本書では、3 の方式を取ります。json で記述されたものを json でエンコードする、というのは一見おかしな感じがしますが、Terraform では正常処理されます。<br>
この方式の場合、 以下のメリットがあります。<br>

+ AWS のドキュメントなどでの json のサンプルをそのままコピペしやすい<br>
+ json 部分にコメントを書ける<br>
+ 最後がカンマで終わっても良い<br>
+ 変数を組み込める<br>

## 3.19.3 repository

repository には、このライフサイクルルールを適用する ECR リポジトリ名を記述します。<br>
aws_ecr_repository.nginx.name と記述することで、その ECR リポジトリの名前が展 開されます。<br>

## 3.19.4 terraform apply の実行<br>

ここまでコードを作成したら、以下を実行してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform apply`を実行<br>

## 3.20 モジュール

nginx イメージ用の aws_ecr_repository と aws_ecr_lifecycle_policy の作成が完了し たので、続いて PHP イメージ用を作成します。<br>
ただ、その記述内容は nginx 用とほぼ同じです。そこで、今回はモジュール化し、同様の 設定の ECR を作成しやすくしてみます。<br>

## 3.20.1 ディレクトリと各種 tf ファイルの作成

まず、モジュールに関するコードの置き場所を決めます。本書では modules というディレクトリを作成し、その配下に配置することにします。<br>
以下の通り、ディレクトリとファイルを作成してください。<br>

+ `laravel-fargate-infra $ mkdir modules && mkdir $_/ecr && touch $_/{main.tf,outputs.tf,variables.tf}`を実行<br>

<img src="https://i.gyazo.com/41a49debd3138679c6160e9ad249d935.png" /> <br>

[Terraform 公式のチュートリアル](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)によれば、モジュールには最低以下の 5 つのファイル を作成することが推奨されています。<br>

• LICENSE : ライセンス情報を記載<br>
• README.md : モジュールの使用方法を記載 • main.tf : 各種リソース等を記述<br>
• variables.tf : 変数を記述<br>
• outputs.tf : 出力を記述<br>

本書では、LICENSE や README.md は割愛し、残り 3 ファイルはたとえ中身が空であっても作成するようにします。<br>

## 3.20.2 main.tfの編集(P45〜)

main.tfを以下の通り編集してください。<br>

+ `laravel-fargate-infra/modules/ecr/main.tf`を編集<br>

```tf:main.tf
resource "aws_ecr_repository" "this" {
  name = var.name

  tags = {
    Name = var.name
  }
}

resource "aws_ecr_lifecycle_policy" "this" {
  policy = jsonencode(
    {
      "rules" : [
        {
          "rulePriority" : 1,
          "description" : "Hold only ${var.holding_count} images, expire all others",
          "selection" : {
            "tagStatus" : "any",
            "countType" : "imageCountMoreThan",
            "countNumber" : var.holding_count
          },
          "action" : {
            "type" : "expire"
          }
        }
      ]
    }
  )

  repository = aws_ecr_repository.this.name
}
```

envs/prod/app/ecr.tf とほぼ同様ですが、以下の点が異なります。<br>

+ Terraform としてのリソース名をベストプラクティスに沿って this としている<br>
+ var.name を使用<br>
+ var.holding_count を使用<br>

__var__<br>

var. 変数名でその変数の値を参照します。変数は後ほど編集する variables.tf 内で宣言します。<br>

__${}__<br>

文字列の中で変数を展開する場合は、${}で囲むようにします。<br>

## 3.20.3 variables.tf の編集

モジュールのベストプラクティスに沿って、変数は variables.tf で宣言するようにします。<br>
以下の通り編集してください。<br>

+ `laravel-fargate-infra/modules/ecr/varibales.tf`を編集<br>

```tf:varibles.tf
variable "name" {
  type = string
}

variable "holding_count" {
  type    = number
  default = 10
}
```

ここでは変数名、型および必要に応じてデフォルト値を宣言し、具体的な値は後ほどモジュールの呼び出し元から与えるようにします。<br>
変数は、以下の形式で宣言します。<br>

```
varibale "変数名" {}
```

__type__<br>

変数の型を宣言します。省略可能ですが、指定することが推奨されています。 [型](https://www.terraform.io/docs/language/expressions/types.html) には以下の種類があります。<br>

+ string<br>
+ number<br>
+ bool<br>
+ list(<TYPE>)<br>
+ set(<TYPE>)<br>
+ map(<TYPE>)<br>
+ object({<ATTR NAME> = <TYPE>, ... }) • tuple([<TYPE>, ...])<br>

__default__<br>

その変数に値が与えられなかった場合のデフォルト値となります。<br>
モジュールの呼び出し元では各変数に値を代入する必要がありますが、デフォルト値が設定されている変数に関してはこれを省略できます。<br>

<img src="https://i.gyazo.com/f9bf5545d44be4e0d5cf7505f1ae98da.png" /> <br>

## 3.20.4 モジュールの呼び出し(P48〜)

続いて、作成したモジュールの呼び出しを行います。ecr.tf を以下の通り編集してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar/ecr.tf`を編集<br>

```tf:ecr.tf
resource "aws_ecr_repository" "nginx" {
  name = "example-prod-foobar-nginx"

  tags = {
    Name = "example-prod-foobar-nginx"
  }
}

resource "aws_ecr_lifecycle_policy" "nginx" {
  policy = jsonencode(
    {
      "rules" : [
        {
          "rulePriority" : 1,
          "description" : "Hold only 10 images, expire all others",
          "selection" : {
            "tagStatus" : "any",
            "countType" : "imageCountMoreThan",
            "countNumber" : 10
          },
          "action" : {
            "type" : "expire"
          }
        }
      ]
    }
  )

  repository = aws_ecr_repository.nginx.name
}

// 追加
module "nginx" {
  source = "../../../../modules/ecr"

  name = "example-prod-foobar-nginx"
}
// ここまで
```

モジュールは以下のように記述することで呼び出せます。<br>

```
module "モジュールにつける名前" {
  source = "モジュールのパス"

  モジュール内で宣言されている変数 = 変数の値 ←デフォルト値が設定されている変数は省略可能
}
```

## 3.20.5 terraform init の実行

モジュールの呼び出しを追加したので、terraform init を実行します (Terraform を初めて触る人にとっては、init というと作成した tfstate や AWS リソースが初期化されてしまうことを想像してしまうかもしれませんが、そういったことは無いので安心して使ってください)。<br>
モジュールの呼び出し元である envs/prod/app/foobar ディレクトリで以下を実行してください。<br>

+ `laravel-fargate/envs/prod/app/foobar $ terraform init`を実行<br>

## 3.20.6 terraform plan の実行

続いて、terraform plan を実行してください。<br>
module.nginx 内で、aws_ecr_lifecycle_policy.this と aws_ecr_repository.this がそれぞれ作成されることが事前確認できます。<br>

+ `laravel-fargate/envs/prod/app/foobar $ terraform plan`を実行<br>

```:terminal
$ terraform plan
aws_ecr_repository.nginx: Refreshing state... [id=example-prod-foobar-nginx]
aws_ecr_lifecycle_policy.nginx: Refreshing state... [id=example-prod-foobar-nginx]

Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # module.nginx.aws_ecr_lifecycle_policy.this will be created
  + resource "aws_ecr_lifecycle_policy" "this" {
      + id          = (known after apply)
      + policy      = jsonencode(
            {
              + rules = [
                  + {
                      + action       = {
                          + type = "expire"
                        }
                      + description  = "Hold only 10 images, expire all others"
                      + rulePriority = 1
                      + selection    = {
                          + countNumber = 10
                          + countType   = "imageCountMoreThan"
                          + tagStatus   = "any"
                        }
                    },
                ]
            }
        )
      + registry_id = (known after apply)
      + repository  = "example-prod-foobar-nginx"
    }

  # module.nginx.aws_ecr_repository.this will be created
  + resource "aws_ecr_repository" "this" {
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

Plan: 2 to add, 0 to change, 0 to destroy.
```

ただし、このまま apply することには問題があります。既に ecr.tf でモジュールを使わずに作成した ECR が存在するところ、module.nginx でさらに ECR を作成することになり、 二重に ECR が作成されてしまうことになります。<br>
ただ、実際のところは AWS の制約により、同じ名前 (今回であれば example-prod-foobar-nginx) の ECR は作成できません。<br>
ですので、apply すると、module.nginx による ECR 作成はエラーになります (plan の段階では、こうした AWS の仕様によるエラーの全てを検出はできません)。<br>
いずれにしても、現状のままではモジュールを使った AWS リソース作成はできません。<br>

## 3.21 terraform state mv (P51〜)


今回のように Terraform で作成済みの AWS リソースを、後からモジュール化するような場合の対応方法を解説します。<br>
Terraform には、state mv というコマンドがあり、tfstate 内のあるリソースの情報を別のリソース情報として移動させることができます。これを利用します。<br>
terraform state mv は以下の形式で実行します。<br>

```
$ terraform state mv 移動先のリソース名 移動先のリソース名
```

## 3.21.1 terraform state mv の実行(P51〜)

まず、aws_ecr_repository.nginx を移動させます。以下のコマンドを実行してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform state mv aws_ecr_repository.nginx module.nginx.aws_ecr_repository.this`を実行<br>

+ 実行後、以下のように表示されれば移動は成功です。<br>

```:terminal
Move "aws_ecr_repository.nginx" to "module.nginx.aws_ecr_repository.this"
Successfully moved 1 object(s).
```

続いて、aws_ecr_lifecycle_policy.nginx を移動させます。以下のコマンドを実行してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform state mv aws_ecr_lifecycle_policy.nginx module.nginx.aws_ecr_lifecycle_policy.this`を実行<br>

実行後、こちらも以下のように表示されれば移動は成功です。<br>

```:terminal
Move "aws_ecr_lifecycle_policy.nginx" to "module.nginx.aws_ecr_lifecycle_policy.this"
Successfully moved 1 object(s).
```

## 3.21.2 不要コードの削除

state mv により、AWS 側に存在する ECR は、モジュールを使った ECR のコードで管理されるようになりました。<br>
モジュールを使っていない ECR のコードは不要になったので、以下の通り削除してください。<br>

<img src="https://i.gyazo.com/cd04957fa441700a0fe72750bf43f6c8.png" /> <br>

+ `laravel-fargate-infra/envs/prod/foobar/ecr.tf`を編集<br>

```tf:ecr.tf
module "nginx" {
  source = "../../../../modules/ecr"

  name = "example-prod-foobar-nginx"
}
```

## 3.21.3 terraform plan により差分が無いことを確認する(P53〜)

state mv の実施と不要なコードの削除により、現在 AWS 側に存在するリソースの状態と、Terraform のコードは一致している (差分が無い) はずです。<br>
確認のため、plan を実行してみましょう。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform plan`を実行<br>

```:terminal
$ terraform plan
module.nginx.aws_ecr_repository.this: Refreshing state... [id=example-prod-foobar-nginx]
module.nginx.aws_ecr_lifecycle_policy.this: Refreshing state... [id=example-prod-foobar-nginx]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and
found no differences, so no changes are needed.
```

plan を実行すると、No changes と表示されており、差分が無いことが確認できました。 ただし、以下が表示されています。<br>

<img src="https://i.gyazo.com/078cd9814aaa6cd2c80130f0649985c8.png" /> <br>

前回実行された terraform apply とは別のところで tfstate に変更が加わっていると、このようなメッセージが出ることがあります。<br>
今回であれば state mv を使ったためです。<br>

## 3.21.4 terraform apply -refresh-only

先ほどの plan 結果の最後を見ると、以下のコマンドが表示されています。<br>

```:terminal
terraform apply -refresh-only
```

このコマンドは、tfstate を現在の AWS リソースの状態に合わせて更新してくれます。 AWS リソース側に変更が加わることはありません。<br>
こちらを実行してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform apply -refresh-only`を実行<br>

```:terminal
$ terraform apply -refresh-only
module.nginx.aws_ecr_repository.this: Refreshing state... [id=example-prod-foobar-nginx]
module.nginx.aws_ecr_lifecycle_policy.this: Refreshing state... [id=example-prod-foobar-nginx]

No changes. Your infrastructure still matches the configuration.

Terraform has checked that the real remote objects still match the result of your
most recent changes, and found no differences.

Would you like to update the Terraform state to reflect these detected changes?
  Terraform will write these changes to the state without modifying any real infrastructure.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value:
```

そのまま、yes を入力してください。入力後、以下が表示されれば問題ありません。<br>

```:terminal
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

もう一度 plan を実行すると以下が表示されます。<br>

```:terminal
$ terraform plan
module.nginx.aws_ecr_repository.this: Refreshing state... [id=example-prod-foobar-nginx]
module.nginx.aws_ecr_lifecycle_policy.this: Refreshing state... [id=example-prod-foobar-nginx]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and
found no differences, so no changes are needed.
```

terraform apply -refresh-only を実行したことにより、先ほどの plan 実行時に表示されていた「Note(以下)」は、今回の plan では表示されなくなりました。<br>

<img src="https://i.gyazo.com/0717c280294fb6ecbf824388302141bd.png" /> <br>
<img src="https://i.gyazo.com/0bcd261a9706813acacc9efc7a0ace46.png" /> <br>

## 3.22 PHP 用 ECR の作成

モジュールを利用して PHP 用の ECR を作成します。ecr.tf を以下の通り編集してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar/ecr.tf`を編集<br>

```tf:ecr.tf
module "nginx" {
  source = "../../../../modules/ecr"

  name = "example-prod-foobar-nginx"
}

// 追加
module "php" {
  source = "../../../../modules/ecr"

  name = "example-prod-foobar-php"
}
// ここまで
```

編集が終わったら、モジュールを追加したので、まず terraform init を実施してください。<br>
その後に、terraform apply を実施してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform init`を実行<br>

+ `laravel-fargate-infra/envs/prod/app/foobar $ terraform apply`を実行<br>

## 3.2.3 Local Values の使用(P57〜)

ECR の作成が完了しましたが、もう少しコードを改善したいと思います。<br>
現状の ecr.tf では、example-prod-foobar という文字列が 2 度出てきます。ここを繰り返 し記述せずに済むようにしてみます。<br>

## 3.23.1 locals.tf の作成と利用

以下の通り、ecr.tf を編集してください。また、locals.tf を新規作成してください。<br>

+ `laravel-fargate-infra/envs/prod/app/foobar/ecr.tf`を編集<br>

```tf:ecr.tf
module "nginx" {
  source = "../../../../modules/ecr"

  name = "example-prod-${local.service_name}-nginx" // 編集
}

module "php" {
  source = "../../../../modules/ecr"

  name = "example-prod-${local.service_name}-php" // 編集
}
```

+ `$ touch laravel-fargate-infra/envs/prod/app/foobar/locals.tf`を実行<br>

+ `laravel-fargate-infra/envs/prod/app/foobar/locals.tf`を編集<br>

```tf:locals.tf
locals {
  service_name = "foobar"
}
```

以上により、${local.service_name}の部分には、foobar が展開されます。example-prod の部分は後ほど対応します。<br>

Local Values は、locals ブロック内に以下の形式で宣言します。<br>

```
locals {
  名前 = 値
  名前 = 値
}
```

## Local Values と変数 (variables) の違い

モジュールの作成の時に登場した変数 (variables) との違いを解説します。<br>
変数はまず名前を宣言しておき、その上で Terraform の実行時に以下の方法で外から値を代入することができます。<br>

+ モジュールの呼び出し元で値を指定する<br>
+ terraform apply 時に-var オプションで指定する<br>
  ◦ terraform apply -var="image_id=ami-abc123"<br>
+ terraform apply 時に tfvars ファイルを読み込ませる<br>
  ◦ terraform apply -var-file="testing.tfvars"<br>
+「TF_VAR_変数名」の環境変数を設定した上で terraform apply する<br>
一方、Local Values は上記のように外から値を代入できません。<br>
Local Values は、繰り返し登場する値を、一箇所で管理するのに役立ちます。<br>

## shared_locals.tf の作成と利用

以下の通り、ecr.tf を編集してください (太字が編集箇所です)。また、locals.tf を新規作成してください。<br>

+ `laravel-fargate-infra/invs/prod/app/foobar/ecr.tf`を編集<br>

```tf:ecr.tf
module "nginx" {
  source = "../../../../modules/ecr"

  name = "${local.name_prefix}-${local.service_name}-nginx" # 編集
}

module "php" {
  source = "../../../../modules/ecr"

  name = "${local.name_prefix}-${local.service_name}-php" # 編集
}
```

+ `$ touch laravel-fargate-infra/envs/prod/shared_locals.tf`を実行<br>

+ `laravel-fargate-infra/envs/prod/shared_locals.tf`を編集<br>

```tf:shared_locals.tf
locals {
  name_prefix = "${local.system_name}-${local.env_name}"
  system_name = "example"
  env_name    = "prod"
}
```

+ `laravel-fargate-infra/envs/prod/app/foobar $ ln -fs ../../shared_locals.tf shared_locals.tf`を実行<br>

<img src="https://i.gyazo.com/bf4c0fc12ef1e1771cb6eb760dd2a3d2.png" /> <br>

上記が完了した後、terraform plan を実施し、差分無し（No changes）となれば問題ありません。<br>

以上で、ECRの作成完了です。次の章では、GitHub Actionsを使って、イメージをビルドしてECRにプッシュします。<br>

<img src="https://i.gyazo.com/1ca29ec1d89384ec69fe9f33328bfd53.png" /> <br>

`*9` https://terragrunt.gruntwork.io/ <br>
