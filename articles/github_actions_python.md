---
title: "Github actionsでpythonからPRにコメントする"
emoji: "🤪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python, githubactions, github]
published: true
---

:::message
[GitHub Actions Advent Calendar 2021](https://qiita.com/advent-calendar/2021/github-actions)の15日目の記事です。
:::

# 何の話
Github actionsで環境変数から値を取得してpythonからPRにコメントする

# はじまり
jsonの比較結果をPRにコメントしようとしてた。
jsonの比較だし、`jq`使ってサクッとコメント用意して、以前同様にcurlでpostすればいいと思っていた。

そしたら、pythonのとあるライブラリが便利なので、それを使って比較しようとなった。
そこまではよかったのだが、pythonからどうやってコメントするんだ…ってなり、
調べてもあまり解答が得られず奮闘する羽目にあった。

備忘録がてら、記事にした。

# どうしたのか
pythonからgithubに触るために、[pygithub](https://pygithub.readthedocs.io/en/latest/introduction.html)を用いることにした。

## workflow
`GITHUB_TOKEN`が必要なため、workflowにて以下のようにenvとして渡す。
また、pythonをセットアップして、`pygithub`をインストールする。

```yaml
name: example

on:
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Set up Python 3.7
        uses: actions/setup-python@v2
        with:
          python-version: 3.7

      - name: Install dependencies
        run: pip install PyGithub

      - name: Example
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: python ./.github/actions/example.py
```

## python
コメントするための関数は以下の通り。
`create_issue_comment`を使っているが、PRにコメントもできる。

```python
from github import Github


def post(message: str):
    """Post message to PR"""
    token = os.getenv("GITHUB_TOKEN")
    repo_name = os.getenv("GITHUB_REPOSITORY")
    with open(os.getenv("GITHUB_EVENT_PATH"), "r") as f:
        github_event = json.load(f)
    pr_number = github_event["number"]

    gh = Github(token)
    repo = gh.get_repo(repo_name)
    pr = repo.get_pull(pr_number)
    pr.create_issue_comment(body=message)
```

## 環境変数
各環境変数を説明すると、

- `GITHUB_TOKEN`: 各ワークフローの実行の開始時に、一意のGITHUB_TOKENシークレットが自動的に作成される。
- `GITHUB_REPOSITORY`: オーナーとレポジトリ名。例えば、`octocat/Hello-World`
- `GITHUB_EVENT_PATH`: Webhookイベントペイロードのパス。詳細情報が入っている。

## コメントの書き方
例えば、f記法の場合は以下のように書ける。
```python
comment = f"""# {result}
<details><summary>show detail</summary>
{detail}
</details>
"""
```

# Ref.

- [automatic token authentication](https://docs.github.com/ja/actions/security-guides/automatic-token-authentication)
- [environment variables](https://docs.github.com/en/actions/learn-github-actions/environment-variables)
