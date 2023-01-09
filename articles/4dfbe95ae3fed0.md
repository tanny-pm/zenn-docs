---
title: "GitHub ActionsでTerraformを自動化する（Terraform Cloudを使わないVer）"
emoji: "🎬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [terraform, githubactions]
published: false
---

# これはなに？

Terraform の公式チュートリアルを参考にして、GitHub Actions で Terraform を自動化した際の学習記録です。チュートリアルでは Terraform Cloud を利用していますが、これを利用しない方法で実装してみました。

参考にした公式チュートリアルはこちらです。事前準備もこちらの記載通りの方法で実施しています。

https://developer.hashicorp.com/terraform/tutorials/automation/github-actions

今回は公式リポジトリをフォークして、オリジナルの実装を追加しました。リポジトリはこちらで公開しています。

https://github.com/tanny-pm/terraform-github-actions

# 公式チュートリアルについて

公式チュートリアルの内容を簡単に紹介しておきます。チュートリアルでは、Pull Request と Merge の実行時に Terraform の各コマンドを実行する GitHub Actions を実装します。

![workflow](https://content.hashicorp.com/api/assets?product=tutorials&version=main&asset=public%2Fimg%2Fterraform%2Fautomation%2Ftfc-gh-actions-workflow.png)

ワークフローの概要は以下のとおりです。

1. フォーマット`fmt`とバリデーションのチェック`validate`を実行する
2. Pull Request の実行時に`plan`を実行し、結果をコメントとして投稿する。
3. main ブランチへのマージ時に`applt`を実行する

チュートリアルでは Terraform の各コマンドの実行を Terraform Cloud を経由して実行しています。ただ、アカウントの作成やセットアップが面倒だったのと、Terraform Cloud を利用する理由がよくわかりませんでした。

そこで今回は、Terraform Cloud を利用せずに、GitHub Actions 上で直接 Terraform コマンドを実行する方法を実装してみました。

# tfstate ファイルの保存先 を S3 に変更する

ここからは、公式チュートリアルの内容から変更した箇所を説明していきます。

公式のチュートリアルでは、`tfstate`ファイルを Terraform Cloud で管理しています。そのため、別の方法で`tfstate`ファイルを管理する必要があります。

:::message
Terraform はインフラの構築状態を`tfstate`ファイルで管理します。これは git ではなく共有ストレージで管理する必要があります。このあたりの説明は[こちらの記事](https://zenn.dev/sway/articles/terraform_staple_sharestate)が詳しいです。
:::

ここでは `tfstate`ファイルを保管する共有ストレージに S3 を利用することにしました。

まず、[こちらの記事](https://blog-benri-life.com/terraform-state-aws-s3-dynamodb-backend/)を参考にして、AWS コンソール画面から S3 バケットを作成しています。（Terraform で作成すると、その`tfstate`ファイルをどうやって管理するか、という問題が永久に発生するため。）

次に、`main.tf`の記述を以下のように変更します。

```diff tf:main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "3.26.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "3.0.1"
    }
  }
  required_version = ">= 1.1.0"

+  backend "s3" {
+  }

-  cloud {
-    organization = "REPLACE_ME"
-
-    workspaces {
-      name = "gh-actions-demo"
-    }
-  }
}
```

本来であればここにバケット名などをハードコーディングするようです。しかし今回は tf ファイルを GitHub のパブリックリポジトリで公開するため、ここにバケット名を記載することは避けました。

Backend の定義には variables を利用できないため、[こちらの記事](https://qiita.com/ymmy02/items/e7368abd8e3dafbc5c52#%E8%A7%A3%E6%B1%BA%E7%AD%962-terraform-init-%E5%AE%9F%E8%A1%8C%E6%99%82%E3%81%AB--backend-config-%E3%81%AB%E3%81%A6%E6%8C%87%E5%AE%9A)を参考にして、`terraform init`実行時に `-backend-config`にてバケット名等を指定します。

以下のように`terraform init`を実行すると、指定したバケットに`tfstate`ファイルが作成されます。

```sh
$ terraform init -backend-config="bucket={{BUKCET NAME}}" \
                 -backend-config="key={{path/to/terraform.tfstate}}" \
                 -backend-config="region=ap-northeast-1"
```

AWS コンソール上からも`tfstate`ファイルを確認できました。
![](https://storage.googleapis.com/zenn-user-upload/a045a4a1100b-20230108.png)

## (任意)AWS のリージョンを東京リージョンに変更する

チュートリアルでは AWS のリージョンが`us-west-2`になっているので、`ap-northeast-1`に変更しておきます。こちらは必要に応じて変更してください。

```diff tf:main.tf
provider "aws" {
+  region = "ap-northeast-1"
-  region = "us-west-2"
}
```

# AWS のクレデンシャル情報をセットアップする

ここからは GitHub Actions の設定を修正していきます。修正した箇所だけ抜粋して紹介します。

まずは AWS のクレデンシャル情報を追記します。[公式のチュートリアル](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions#set-up-terraform-cloud)では Terraform Cloud の環境変数に AWS の ID とアクセスキーを記載しています。今回はこれを GitHub Actions の YML ファイルに記載...しようと思ったのですが、OIDC を利用した認証の方がよりセキュアということなので、その方法を採用しました。

具体的な方法は、以下の記事をそのまま参考にさせていただきました。
https://zenn.dev/kou_pg_0131/articles/gh-actions-oidc-aws

YML ファイルの該当箇所だけ抜き出すと、以下のようになります。

```yml:terraform.yml
    permissions:
      id-token: write
      contents: read
# 省略
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@master
        with:
          aws-region: ${{ env.AWS_DEFAULT_REGION }}
          role-to-assume: ${{ env.AWS_ROLE_ARN }}
```

これで、 GitHub Actions 上で AWS のリソース作成等を実行できるようになりました。

# Terraform をセットアップする

次に Terraform のセットアップ処理を修正します。`hashicorp/setup-terraform`アクションの[公式ドキュメント](https://github.com/marketplace/actions/hashicorp-setup-terraform#usage)によると、`cli_config_credentials_token`が設定されていれば Terraform コマンドの実行時に Terraform Cloud を利用するようです。設定しない場合は Terraform CLI を使います。

今回は以下のように、`cli_config_credentials_token`を削除すれば設定完了です。ついでに Terraform のバージョンも指定しておきました。

```yml:terraform.yml
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: ${{ env.TF_VERSION }}
          # cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}
```

# init コマンドを実行する

`terraform init`を実行する箇所では、先ほど紹介したように、Backend の設定をオプションで指定します。S3 のバケット名は Secrets に設定した値を呼び出しています。

```yml:terraform.yml
      - name: Terraform Init
        id: init
        run: >
          terraform init -backend-config="bucket=${{ env.S3_BACKEND }}"
          -backend-config="key=state/terraform.tfstate"
          -backend-config="region=${{ env.AWS_DEFAULT_REGION }}"
```

これで、S3 に保存された`tfstate`ファイルを参照するようになります。

# Pull Request にコメントを投稿する

[公式チュートリアル](https://content.hashicorp.com/api/assets?product=tutorials&version=main&asset=public%2Fimg%2Fterraform%2Fautomation%2Fgh-actions-pr-plan.gif)では、`terraform plan`などの実行結果を Pull Request のコメントに投稿する機能も実装されています。

![Pull Request](https://content.hashicorp.com/api/assets?product=tutorials&version=main&asset=public%2Fimg%2Fterraform%2Fautomation%2Fgh-actions-pr-plan.gif)

ただし、ここまでの設定のままで GitHub Actions を実行すると、エラーが出てしまいます。どうやら AWS の OIDC の設定時に Permissions の設定を追記したことが影響しているようです。

以下の記事で詳しく説明されていたので、こちらを参考にして`pull-requests`のパーミッション設定を追記しました。
https://sadayoshi-tada.hatenablog.com/entry/2022/04/09/115740

```yml:terraform.yml
    permissions:
      id-token: write
      contents: read
      pull-requests: write # 追記
```

これで Pull Request にコメントが投稿されるようになりました！（ついでに GitHub Actions の Pesmissions の勉強にもなりました。）

# Pull Request と Merge を実行する

ここまでの設定により、Pull Request と Merge の実行時に指定したアクションが実行されるようになります。

[公式チュートリアル](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions#review-and-merge-pull-request)の実行結果と同じ結果となるため、具体的な結果はそちらを参考にしてください。

参考として、初回の`apply`実行の後に、`main.tf`ファイルの中身を変えずに再度 Merge した後の実行結果を紹介します。以下のように、S3 の`tfstate`ファイルを参照して、既存の構築内容と差分がないことを検知し、何も実行していません。

![](https://storage.googleapis.com/zenn-user-upload/58d8a1aacee7-20230109.png)

また、if 分岐（`if: github.event_name == 'pull_request'`）により、Merge の実行時には`plan`の実行がスキップされていることもわかります。if 分岐を使わずに Pull request と Merge で YML ファイルを分ける実装例もあるようですが、このくらいの記述であれば、1 つの YML ファイルに記載した方がわかりやすいと感じました。

# おわりに

Terraform Cloud を利用するのは面倒だという思いから、今回は GitHub Actions のみで Terraform の実行を自動化する方法を試してみました。その結果、Terraform Cloud を利用する理由やメリットも理解することができました。

今回は`tfstate`ファイルを保存するために S3 を利用し、AWS の OIDC 設定を GitHubActions 上に記載し直すことになりました。Terraform Cloud を利用すれば、このあたりの「Terraform の実行に必要な情報」の管理をおまかせすることができますね。GitHub Actions 側には Terraform コマンドの実行手順だけを書けば良いです。

実務で利用する際にはこのあたりのメリデメを考慮して、利用するツールを選定することになりそうです。

---

# 参考文献

https://developer.hashicorp.com/terraform/language/settings/backends/s3

https://zenn.dev/makumattun/articles/bf47833e9d062d

https://zenn.dev/shunsuke_suzuki/articles/improve-cicd-with-github-comment
