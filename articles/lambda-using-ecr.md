---
title: "terraformによりlambdaとしてコンテナイメージをデプロイ"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [terraform, aws, lambda, ecr]
published: true
---

# コンテナイメージを用いたlambda

## 概要

- 2020/12からlambdaとしてECR上のコンテナイメージをデプロイできる
- 今までのzip形式との大きな違いはアーティファクトサイズ制限(250 MB -> 10 GB)
- ローカルでテストするのが簡単 ([Lambda コンテナイメージをローカルでテストする](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/images-test.html))

## デプロイ方法
zipとコンテナイメージの両方を簡単な図で示すと以下のようになる。

![](https://storage.googleapis.com/zenn-user-upload/aff378b02885523935513230.png)

# terraformによるデプロイ

terraformによりlambdaの作成のみを行い、作成後のイメージ管理は行わない。
この場合、github actions等を用いて、イメージのbuild, push, lambdaの更新までを行うと管理が楽になる。

```
resource "aws_lambda_function" "function" {
  function_name = "<Function name>"
  description   = "<Description>"

  package_type = "Image"
  image_uri    = "<Account id>.dkr.ecr.<Region>.amazonaws.com/<ECR>:<Tags>"
  timeout      = <Timeout>

  lifecycle {
    ignore_changes = [image_uri]
  }

  role = <Role arn>
}
```

terraformでイメージ管理を行わない場合、イメージ更新の際は`update-function-code`を用いると良い。
コマンド例は以下の通り。

```sh
aws lambda update-function-code --function-name "<Function name>" --image-uri "<Account id>.dkr.ecr.<Region>.amazonaws.com/<ECR>:<Tags>"
```

# 気づき

## パブリックイメージは使用できない

パブリックイメージは使用できず、自アカウントのイメージを指定する必要がある。
このことに気づいたのは、ダミーとして適当なイメージを使おうとして、パブリックイメージが指定できなかったためである。
terraformによりイメージを管理しないと、lambda生成時にはダミーイメージを用いたら以下の二点が便利である。

- 簡素なmoduleが用意できる (必要な引数は`function_name, description, role`で、あとはデフォルト値を設定)
- lambdaとecrを一度に作成できる (ダミーを使わない場合、lambdaを作る前にecrを作り、イメージをpushしておく必要がある)


### サクッとダミーイメージを用意する

そこでダミーイメージを簡単に作れるようにした ([yoshi65/lambda_hello_world_image](https://github.com/yoshi65/lambda_hello_world_image.git))。
以下の手順で、ecrに`dummy:latest`が生成できる。事前に、認証情報は設定しておく必要がある。

```sh
git clone https://github.com/yoshi65/lambda_hello_world_image.git
cd lambda_hello_world_image
./hello_world.sh
```

もし消したくなったら、以下の通り。
```sh
./hello_world.sh -d
```

この生成に、terraformを使うか考えたけど、

- ダミーイメージ自体はコンソールやcliから作る際にも活用できる
- ecrをterraformで作ったところで、結局イメージプッシュのためのスクリプトは必要

などにより、使わなかった (気が向いた時に追加するかも)。

## `update-function-code`後、すぐ`invoke`できない

lambdaを使用する際に、`$LATEST`を使わず、aliasを指定して、`invoke`することは多い。
そこで、動作確認のために`invoke`し、正常終了を確認してから、aliasを更新しようとした。

その時、`invoke`において

- `$LATEST`を指定すると前のバージョンが使われる
- `update-function-code`でパブリッシュされたバージョンを使うと`ResourceConflictException`が生じる

### `invoke`できないわけ
この理由としては、イメージの最適化が挙げられる。
コンソール上でイメージ更新を確認したことがあればわかると思うが、更新後すぐは更新中といった旨が上部に表示され、しばらくしたら完了の旨が表示される。

### `invoke`するために
解決法としては、ステータスの更新を待てばいい。
以下のコマンドで解決する。
```sh
aws lambda wait function-updated --function-name "<Function name>"
```

lambdaの更新、待機、起動をまとめると、
```sh
aws lambda update-function-code --function-name "<Function name>" --image-uri "<Image uri>" --publish
aws lambda wait function-updated --function-name "<Function name>"
aws lambda invoke --function-name "<Function name>":\$LATEST response.json
```

# 感想

好みはありそうだが、コンテナイメージの方がzipより使い勝手はよかった。
コンテナイメージで設定していることが多いため、lambdaを作る際に必要な設定が減る。

# Ref.

- [コンテナ利用者に捧げる AWS Lambda の新しい開発方式 !](https://aws.amazon.com/jp/builders-flash/202103/new-lambda-container-development/?awsf.filter-name=*all)
- [New for AWS Lambda – Container Image Support](https://aws.amazon.com/jp/blogs/aws/new-for-aws-lambda-container-image-support/)
- [Lambda コンテナイメージをローカルでテストする](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/images-test.html)
