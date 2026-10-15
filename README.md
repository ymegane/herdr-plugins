# herdr-plugins

個人用の [herdr](https://herdr.dev) プラグイン置き場。プラグインごとにサブディレクトリを切っている。

| プラグイン | 概要 |
|---|---|
| [jira-pr](./jira-pr/) | エージェントpaneごとに、Jiraのステータス、GitHub PR、worktreeを Agentsサイドバーとポップアップの一覧に表示する |

## 導入

herdrのGitHubインストールで入れる。

```bash
herdr plugin install ymegane/herdr-plugins/jira-pr
```

作者はクローンしたディレクトリを `herdr plugin link` で登録して使っており、`plugin install` での導入は試していない。手元で編集しながら使うなら、次のように登録する。

```bash
git clone https://github.com/ymegane/herdr-plugins.git ~/Developer/github/herdr-plugins
herdr plugin link ~/Developer/github/herdr-plugins/jira-pr
```

`link` はファイルをコピーしないので、作業ツリーの変更は次の呼び出しからそのまま反映される。プラグインごとの設定は各ディレクトリの README を参照。

## 動作要件

- 開発と動作確認は macOS と herdr 0.9.5 でのみ行っている
- スクリプトは macOS 同梱の bash 3.2 で動く書き方にしてある
