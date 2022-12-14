## 2.4.2 IAMユーザーのアクセスキーの発行(P13〜)

AWS CLI で、ある IAM ユーザーの権限で操作するには、その IAM ユーザーのアクセスキー ID とシークレットアクセスキーが必要です。これを発行します。<br>

もし、AWS CLI を普段から使っていて、AdministratorAccess 権限を持った IAM ユー ザーをプロファイルに設定済みであれば実施することはありません。<br>
「2.5 tfstate 管理用の S3 バケットの作成」(p.19) に飛んで、そこから先を読んでください。<br>
AWS CLI を使うのが初めてであったり、プロファイルというものをよく知らないという 方は、このまま読み進めてください。<br>

__IAMの選択__<br>

ブラウザから AWS マネジメントコンソールの IAM に移動してください。<br>

もし、AWS CLI を普段から使っていて、AdministratorAccess 権限を持った IAM ユーザーをプロファイルに設定済みであれば実施することはありません。<br>
「2.5 tfstate 管理用の S3 バケットの作成」(p.19) に飛んで、そこから先を読んでください。<br>
AWS CLI を使うのが初めてであったり、プロファイルというものをよく知らないという方は、このまま読み進めてください。<br>

## IAMの選択

ブラウザから AWS マネジメントコンソールの IAM に移動してください。<br>

<img src="https://i.gyazo.com/74d9fb7f06eb8fc1d0acc5f511983d8d.png" alt="aws_iam_setting" title="aws_iam"> <br>

上の画面では、terraform という名前の IAM ユーザーを選択していますが、これは筆者 の AWS アカウント内だけに存在する、AdministratorAccess 権限を持った IAM ユーザーです。実際には、あなたの AWS アカウント内に存在する AdministratorAccess 権限を持っている IAM ユーザーを選択するようにしてください。<br>

`＊` 参考: [管理者権限をもつIAMユーザを作成](https://zenn.dev/mo_ri_regen/articles/aws-iam-with-administrator-rights) <br>

## アクセスキーの作成

「認証情報」タブを選択すると表示される、「アクセスキーの作成」ボタンを押してください。<br>

<img src="https://i.gyazo.com/ea132accb6ca767758afbc857b2ccf0d.png" alt="aws_iam_access_key" title="iam_key"> <br>

以下の画面が表示され、アクセスキー ID とシークレットアクセスキーが確認できるようになります。<br>

<img src="https://i.gyazo.com/ec9c019a65d0b8d2e5a50b59c7495c7e.png" alt="iam_access_key" title="サンプル"> <br>

この画面は閉じずにそのままにしておいてください。<br>

+ `注意!`<br>

アクセスキーID とシークレットアクセスキーの値は他人に知られないよう注意してください。<br>
例えば、何かのファイルにメモ代わりに記述して、気づかないうちにうっかり GitHub のパブリックリポジトリにプッシュしてしまった、といったようなことが無いようにしてください。<br>
もし、他人に知られてしまうと、悪用されて高額課金されてしまうというリスクがあります。<br>

## 2.4.3 AWS CLIでのアクセスキーIDとシークレットアクセスキーの利用方法について

先ほどのアクセスキー ID とシークレットアクセスキーを使えるように AWS CLI に設定すると、AdministratorAccess 権限でターミナルから AWS の操作を行うことができます。<br>
具体的な設定方法としては、環境変数に設定する方法と、プロファイルに設定する方法があります。環境変数に設定する場合は、ターミナルで以下のようにコマンドを実行します。<br>

```:terminal
$ export AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXXXXXX
$ export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
$ export AWS_REGION=ap-northeast-1
$ export AWS_PROFILE=terraform
```

もうひとつの設定方法であるプロファイルを利用する場合は、上記のような一連の設定に名前を付けて呼び出すことができます。<br>
プロファイルを利用した方が管理しやすいので、本書ではプロファイルの設定方法を説明します。<br>

## 2.4.4 プロファイルの作成と設定

まず、プロファイルを作成します。ターミナルで以下コマンドを実行してください (実行 場所のディレクトリはどこでも構いません)。<br>
terraform の部分は、プロファイルに付ける名前です。terraform 以外のお好みの名前でも構いません。<br>

```:terminal
$ aws configure --profile terraform
```

## アクセスキーIDの設定

コマンドを実行すると、最初に以下が表示されます。<br>

```:terminal
AWS Access Key ID [None]:
```

以下画面のアクセスキー ID の値を貼り付けてください。<br>

<img src="https://i.gyazo.com/95d0f5b9267c9afde011b9f2e0ba4b88.png" alt="iam_access_key" title="サンプル"> <br>

以下画面の表示を押すと表示される、シークレットアクセスキーの値を貼り付けてください。<br>

<img src="https://i.gyazo.com/79aa48bdc487370d2851094738d523d4.png" alt="iam_access_key" title="サンプル"> <br>

入力が終わったら、ターミナルでエンターキーを押してください。<br>

## リージョンの設定

次に以下が表示されます。<br>

```:terminal
Default region name [None]:
```

ここでは以下のように ap-northeast-1(アジアパシフィック東京) と入力してエンターキーを押してください。<br>

```:terminal
Default region name [None]: ap-northeast-1
```

## 出力フォーマットの設定

最後に以下が表示されます。<br>

```:terminal
Default output format [None]:
```

ここでは以下のように json と入力してエンターを押してください。<br>

```:terminal
Default output format [None]: json
```

ここで設定したのは、AWS CLI を実行した結果の出力形式です。他にも text などが選択*1 できます。<br>

`*1` https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-usage-output.html <br>

## プロファイルの確認

以上で、一連の設定をひとまとめにしたプロファイルが新規作成されました。この段階で はプロファイルが作成されただけで、まだ使用する状態になっていません。<br>
作成したプロファイルを使用するには、環境変数 AWS_PROFILE にプロファイル名を設定します。<br>
以下コマンドを実行してください (以下は、プロファイルの名前を terraform で作成した 場合の例です)。<br>

```:terminal
$ export AWS_PROFILE=terraform
```

次に、以下コマンドを実行してください。<br>

```:terminal
$ aws configure list
```


現在使用中のプロファイルが、表形式で表示されます。<br>

```:terminal
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                terraform              env    ['AWS_PROFILE', 'AWS_DEFAULT_PROFILE']
access_key     ****************BV6O              env
secret_key     ****************Ml1r              env
    region           ap-northeast-1              env    ['AWS_REGION', 'AWS_DEFAULT_REGION']
```

profile と書かれた行の Value 列の値が、先ほど作成したプロファイルの名前 (terraform 等) となっていれば問題ありません。<br>
今後、本書で Terraform を使っていく中で、もしも権限不足などでエラーになっていると思われる時は、aws configure list コマンドを使って、正しいプロファイルを使用する設定になっているかどうかを確認するようにしてください。<br>

## 2.5 tfstate 管理用の S3 バケットの作成(P19〜)

Terraform では、インフラの最新の状態を tfstate と呼ばれるファイルで管理します。<br>
この tfstate はデフォルトではローカルに保存されますが、チームで Terraform を扱う場合は、ローカルではなくチーム全員が参照・更新可能な場所に保存する必要があります。<br>

読者の方は1人で本書のチュートリアルを進めていくでしょうから、tfstate をローカルに保存して進めることも可能かと思いますが、チーム開発などにも対応できるよう、tfstate を AWS のストレージサービスである S3 に保存していくようにします。<br>
なお、今回対応は見送りますが、Terraform では複数人での tfstate の更新が競合しないよう、DynamoDB を使ってロックを行う*2こともできます。<br>

`＊2` https://www.terraform.io/docs/language/settings/backends/s3.html%23dynamodb-state-locking <br>

## 2.5.1 S3 バケットの作成(P20〜)

S3 バケットは先ほど設定が完了した AWS CLI を使って数回のコマンドで作成することもできるのですが、ここではマネジメントコンソールから作成していくことにします。<br>

## バケットの作成の選択

ブラウザから AWS マネジメントコンソールの S3*3に移動し、「バケットの作成」を選択 してください。<br>

+ `＊3` https://s3.console.aws.amazon.com/s3/home?region=ap-northeast-1 <br>

<img src="https://i.gyazo.com/578176c80319bd37701143b6d1a9e077.png" alt="iam_access_key" title="サンプル"> <br>

## バケット名の入力とリーションの選択

以下の画面が表示されるので、バケット名を入力し、リージョンを選択します。<br>

<img src="https://i.gyazo.com/b9a20e8f9706ce6040395c770bfd8c03.png" alt="iam_access_key" title="サンプル"> <br>

バケット名は、世界中で他と被らないユニークな名前にする必要があります。<br>
画面例では、筆者の SNS などでのアカウント名を含めるようにして、shonansurvivors-tfstate としました。<br>
読者の方は、何かユニークな名前-tfstate とするようにしてください。<br>
リージョンについては、「アジアパシフィック (東京) ap-northeast-1」を選択してください。<br>

## ブロックパブリックアクセスのバケット設定

「パブリックアクセスをすべてブロック」はチェックをしたままとします。<br>

<img src="https://i.gyazo.com/bda315c26de0f19966b18bdeb874176d.png" alt="iam_access_key" title="サンプル"> <br>

## バージョニング

「バケットのバージョニング」は「有効にする」を選択してください。<br>

<img src="https://i.gyazo.com/49efb0c4246c2ca965a651b43eed7cc0.png" alt="iam_access_key" title="サンプル"> <br>

バージョニング*4を有効化すると、S3 バケット内のオブジェクト (ファイル) を世代管理します。<br>
バージョニングを設定しておくことで、tfstate を誤って削除したり更新したりしてしまった時のリカバリが可能となります。<br>

+ `＊4` https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/dev/Versioning.html <br>

## 暗号化

「サーバー側の暗号化」は「有効にする」、「暗号化キータイプ」は「Amazon S3 キー (SSE-S3)」を選択してください。<br>

<img src="https://i.gyazo.com/1daba7c9358994213366e6e70ba736d2.png" alt="iam_access_key" title="サンプル"> <br>

## バケットの作成

全ての入力が終わったら、「バケットを作成」を押してください。<br>

<img src="https://i.gyazo.com/34af38878059e603b904eaa7a47d6e0f.png" alt="iam_access_key" title="サンプル"> <br>

以上で本章は完了です。次の章では、Terraform を使って、AWS リソースを作成していきます。<br>
