---
title: "github actions OIDCとs3 bucket policyの例外指定にハマった"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [terraform, aws, iam]
published: false
---

# 概要

github actionsでのAWS認証をIAM userからrole for OIDCに移行する際に、s3 bucket policyの例外指定にハマった

# なんで移行するの

- userを使用する場合、AWS認証情報をsecretとして扱う必要があり、以下のような課題があったため
  - secret漏洩リスク
  - 漏洩リスクを考慮したローテーションの手間
  - userを意図しない用途で使用できてしまう

# 何をしたのか

## userをroleに置換
1. 新しいroleを作成
1. userにattachしていたpolicyをroleにattach
1. s3 bucket policyのnot principalをuserからroleに更新

s3 bucket policyを例示すると、userを使用していた時は以下。
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Deny",
        "NotPrincipal": {"AWS": [
            "arn:aws:iam::444455556666:user/Bob",
            "arn:aws:iam::444455556666:root"
        ]},
        "Action": "s3:*",
        "Resource": [
            "arn:aws:s3:::BUCKETNAME",
            "arn:aws:s3:::BUCKETNAME/*"
        ]
    }]
}
```
roleを使用するために書き換えた後は以下。
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Deny",
        "NotPrincipal": {"AWS": [
            "arn:aws:iam::444455556666:role/cross-account-read-only-role",
            "arn:aws:iam::444455556666:root"
        ]},
        "Action": "s3:*",
        "Resource": [
            "arn:aws:s3:::Bucket_AccountAudit",
            "arn:aws:s3:::Bucket_AccountAudit/*"
        ]
    }]
}
```

## github actionsを更新
- github actionsでの認証を[Configure AWS Credentials](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions)に更新

userを使用していた時は以下のように環境変数としてAWS認証情報をstepに渡していた。
```yaml
jobs:
  workflow:
    steps:
      - name: Confirm s3 bucket
        run: aws s3 ls s3://bucket-name
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}
```
roleを使用するために以下のように認証のstepを追加した。
```yaml
jobs:
  workflow:
    steps:
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::444455556666:role/cross-account-read-only-role
          role-session-name: cross-account-audit-app
          aws-region: AWS_REGION
```

## なぜか`Access Denied`
- user使用時と同じpolicyを使用しているので、permission不足等の可能性はなさそう
- s3 bucket policyの方に問題があると判断

## ドキュメントを読む
- [AWS JSON policy elements: NotPrincipal](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notprincipal.html)を確認すると、`assumed-role`もnot principalに含める必要があった

つまり、適切なs3 bucket policyは以下のようになる。
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Deny",
        "NotPrincipal": {"AWS": [
            "arn:aws:sts::444455556666:assumed-role/cross-account-read-only-role/cross-account-audit-app",
            "arn:aws:iam::444455556666:role/cross-account-read-only-role",
            "arn:aws:iam::444455556666:root"
        ]},
        "Action": "s3:*",
        "Resource": [
            "arn:aws:s3:::Bucket_AccountAudit",
            "arn:aws:s3:::Bucket_AccountAudit/*"
        ]
    }]
}
```

# 気づき
- 当たり前のことだけど、ドキュメントをちゃんと読みましょうってお話。

# Ref.
1. [Configure AWS Credentials](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions)
1. [AWS JSON policy elements: NotPrincipal](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notprincipal.html)
