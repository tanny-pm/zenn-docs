---
title: "GitHub ActionsでTerraformを自動化する（Terraform Cloudを使わないVer）"
emoji: "🎬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [terraform, githubactions]
published: true
---

# これはなに？

Terraformの公式チュートリアルを参考にして、GitHub ActionsでTerraformを自動化した際の学習記録です。チュートリアルではTerraform Cloudを利用していますが、これを利用しない方法で実装してみました。

参考にした公式チュートリアルはこちらです。事前準備もこちらの記載通りの方法で実施しています。

https://developer.hashicorp.com/terraform/tutorials/automation/github-actions

今回は公式リポジトリをフォークして、オリジナルの実装を追加しました。リポジトリはこちらで公開しています。

https://github.com/tanny-pm/terraform-github-actions

---

# 公式チュートリアルについて

はじめに、公式チュートリアルの内容を簡単に紹介しておきます。チュートリアルでは、Pull RequestとMergeの実行時にTerraformの各コマンドを実行するGitHub Actionsを実装します。

![workflow](https://content.hashicorp.com/api/assets?product=tutorials&version=main&asset=public%2Fimg%2Fterraform%2Fautomation%2Ftfc-gh-actions-workflow.png)

ワークフローの概要は以下のとおりです。

1. フォーマット`fmt`とバリデーションのチェック`validate`を実行する。
2. Pull Requestの実行時に`plan`を実行し、結果をコメントとして投稿する。
3. mainブランチへのマージ時に`applt`を実行する。

チュートリアルではTerraformの各コマンドをTerraform Cloudを経由して実行しています。ただ、アカウントの作成やセットアップが面倒で、なおかつ（最初にチュートリアルを見た時は）Terraform Cloudを利用する理由がよくわかりませんでした。

そこで今回は、Terraform Cloudを利用せずに、GitHub Actions上で直接Terraformコマンドを実行する方法を実装してみました。

# tfstate ファイルの保存先 を S3 に変更する

ここからは、公式チュートリアルに変更を加えた箇所を説明します。

公式チュートリアルでは、`tfstate`ファイルをTerraform Cloudで管理しています。そのため、別の方法で`tfstate`ファイルを管理する必要があります。

:::message
Terraformはインフラの構築状態を`tfstate`ファイルで管理します。これはgitではなく共有ストレージで管理する必要があります。このあたりの説明は[こちらの記事](https://zenn.dev/sway/articles/terraform_staple_sharestate)が詳しいです。
:::

今回は`tfstate`ファイルを保管する共有ストレージとしてS3を利用することにしました。

まず、[こちらの記事](https://blog-benri-life.com/terraform-state-aws-s3-dynamodb-backend/)を参考にして、AWSコンソール画面からS3バケットを作成します。（Terraformで作成すると、その`tfstate`ファイルをどうやって管理するか、という問題が永久に発生するため。）

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

本来であれば`backend`の箇所にバケット名などをハードコーディングするようです。しかし今回はtfファイルをGitHubのパブリックリポジトリで公開するため、ここにバケット名を記載することは避けました。

Backendの定義にはvariablesを利用できないため、[こちらの記事](https://qiita.com/ymmy02/items/e7368abd8e3dafbc5c52#%E8%A7%A3%E6%B1%BA%E7%AD%962-terraform-init-%E5%AE%9F%E8%A1%8C%E6%99%82%E3%81%AB--backend-config-%E3%81%AB%E3%81%A6%E6%8C%87%E5%AE%9A)を参考にして、`terraform init`実行時に `-backend-config`にてバケット名等を指定します。以下のように`terraform init`を実行すると、指定したバケットに`tfstate`ファイルが作成されます。

```sh
$ terraform init -backend-config="bucket={{BUKCET NAME}}" \
                 -backend-config="key=state/terraform.tfstate" \
                 -backend-config="region=ap-northeast-1"
```

AWSコンソール上からも`tfstate`ファイルを確認できました。
![](https://storage.googleapis.com/zenn-user-upload/a045a4a1100b-20230108.png)

## (任意)AWS のリージョンを東京リージョンに変更する

チュートリアルではAWSのリージョンが`us-west-2`になっているので、`ap-northeast-1`に変更しておきます。こちらは必要に応じて変更してください。

```diff tf:main.tf
provider "aws" {
+  region = "ap-northeast-1"
-  region = "us-west-2"
}
```

# AWS のクレデンシャル情報をセットアップする

ここからはGitHub Actionsの設定を修正します。修正した箇所だけ抜粋して紹介します。

まずはAWSのクレデンシャル情報を追記します。[公式のチュートリアル](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions#set-up-terraform-cloud)ではTerraform Cloudの環境変数にAWSのIDとアクセスキーを記載しています。今回はこれをGitHub ActionsのYMLファイルに記載しようと思ったのですが、OIDCを利用した認証の方がよりセキュアということなので、その方法を採用しました。

具体的な方法は、以下の記事をそのまま参考にさせていただきました。
https://zenn.dev/kou_pg_0131/articles/gh-actions-oidc-aws

YMLファイルの該当箇所だけ抜き出すと、以下のようになります。

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

これで、 GitHub Actions上でAWSのリソース作成等を実行できるようになりました。

# Terraform をセットアップする

次にTerraformのセットアップ処理を修正します。`hashicorp/setup-terraform`アクションの[公式ドキュメント](https://github.com/marketplace/actions/hashicorp-setup-terraform#usage)によると、`cli_config_credentials_token`が設定されていればTerraformコマンドの実行時にTerraform Cloudを利用するようです。設定しない場合はTerraform CLIを使います。

今回は以下のように、`cli_config_credentials_token`を削除すれば設定完了です。ついでにTerraformのバージョンも指定しておきました。

```yml:terraform.yml
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: ${{ env.TF_VERSION }}
          # cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}
```

# init コマンドを実行する

`terraform init`を実行する箇所では、先ほど紹介したように、Backendの設定をオプションで指定します。S3のバケット名はSecretsに設定した値を呼び出しています。

```yml:terraform.yml
      - name: Terraform Init
        id: init
        run: >
          terraform init -backend-config="bucket=${{ env.S3_BACKEND }}"
          -backend-config="key=state/terraform.tfstate"
          -backend-config="region=${{ env.AWS_DEFAULT_REGION }}"
```

これで、S3に保存された`tfstate`ファイルを参照するようになります。

# Pull Request にコメントを投稿する

[公式チュートリアル](https://content.hashicorp.com/api/assets?product=tutorials&version=main&asset=public%2Fimg%2Fterraform%2Fautomation%2Fgh-actions-pr-plan.gif)では、`terraform plan`などの実行結果をPull Requestのコメントに投稿する機能も実装されています。これにより、レビュアーはコメントを読むだけでインフラの変更内容を確認できます。

![Pull Request](https://content.hashicorp.com/api/assets?product=tutorials&version=main&asset=public%2Fimg%2Fterraform%2Fautomation%2Fgh-actions-pr-plan.gif)

ただし、ここまでの設定のままでGitHub Actionsを実行すると、エラーが出てしまいます。どうやらAWSのOIDCの設定時にPermissionsの設定を追記したことが影響しているようです。

以下の記事で詳しく説明されていたので、こちらを参考にして`pull-requests`のパーミッション設定を追記しました。
https://sadayoshi-tada.hatenablog.com/entry/2022/04/09/115740

```yml:terraform.yml
    permissions:
      id-token: write
      contents: read
      pull-requests: write # 追記
```

これでPull Requestにコメントが投稿されるようになりました！（ついでにGitHub ActionsのPesmissionsの勉強にもなりました。）

# Pull Request と Merge を実行する

ここまでの設定により、Pull RequestとMergeの実行時に指定したアクションが実行されるようになります。実行結果は[公式チュートリアル](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions#review-and-merge-pull-request)の結果と同じになるため、具体的な内容はそちらを見てください。

参考として、初回の`apply`実行の後に、`main.tf`ファイルの中身を変えずに再度Mergeした後の実行結果を紹介します。以下のように、S3の`tfstate`ファイルを参照して、既存の構築内容と差分がないことを検知し、インフラは変更されていません。

![](https://storage.googleapis.com/zenn-user-upload/58d8a1aacee7-20230109.png)

また、if分岐（`if: github.event_name == 'pull_request'`）により、Mergeの実行時には`Terraform Plan`セクションの実行がスキップされていることもわかります。（Pull Requestの時は`Terraform Apply`の実行がスキップされます`）if分岐を使わずにPull requestとMergeでYMLファイルを分ける実装例もあるようですが、このくらいの記述であれば、1つのYMLファイルに記載した方がわかりやすいと感じました。

# おわりに

Terraform Cloudを利用するのは面倒だという思いから、今回はGitHub ActionsのみでTerraformの実行を自動化する方法を試してみました。その結果、Terraform Cloudを利用する理由やメリットも理解することができました。

今回は`tfstate`ファイルを保存するためにS3を利用し、AWSのOIDC設定をGitHubActions上へ記載し直すことになりました。Terraform Cloudを利用すれば、このあたりの「Terraformの実行に必要な情報」の管理をCloudにおまかせできます。この場合、GitHub Actions側にはTerraformコマンドの実行手順だけを書けば良いです。

実務で利用する際にはこのあたりのメリデメを考慮して、利用するツールを選定することになりそうです。

---

# 参考文献

https://developer.hashicorp.com/terraform/language/settings/backends/s3

https://zenn.dev/makumattun/articles/bf47833e9d062d

https://zenn.dev/shunsuke_suzuki/articles/improve-cicd-with-github-comment
