# herdr-plugins

個人用の [herdr](https://herdr.dev) プラグイン置き場。プラグインごとにサブディレクトリを切っている。

| プラグイン | 概要 |
|---|---|
| [jira-pr](./jira-pr/) | エージェントpaneごとに、Jiraのステータス、GitHub PR、worktreeを Agentsサイドバーに表示する |

## 導入

プライベートリポジトリなので、クローンしたディレクトリを `herdr plugin link` で登録する。

```bash
git clone git@github.com:ymegane/herdr-plugins.git ~/Developer/github/herdr-plugins
herdr plugin link ~/Developer/github/herdr-plugins/jira-pr
```

`link` はファイルをコピーしないので、作業ツリーの変更は次の呼び出しからそのまま反映される。プラグインごとの設定は各ディレクトリの README を参照。

## 動作要件

- 開発と動作確認は macOS と herdr 0.9.5 でのみ行っている
- スクリプトは macOS 同梱の bash 3.2 で動く書き方にしてある
