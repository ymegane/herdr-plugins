# jira-pr

エージェントが動いているpaneごとにJiraキーを特定し、Jiraのステータス、GitHub PR、worktreeを herdrの Agentsサイドバーに表示する。

```
✳ myapp  main
ABC-123-fix-login-redirect
claude
[Jira] ABC-123 レビュー中                ← 青（進行中）
[PR] #456 · [CI成功] · [レビュー待ち]    ← 緑 · 緑 · 黄
[branch] fix-login-redirect              ← 薄色
```

`[ ]` の部分はNerd Fontのアイコンで表示される。状態は色でも区別する。

## 紐付けの規則

プラグインは次の順でpaneのJiraキーを決める。

1. paneの `foreground_cwd` のブランチ名にキーがあれば、それを使う。PRはそのブランチで引き、worktreeはそのチェックアウトになる
2. ブランチ名にキーが無ければ、paneのラベルとターミナルタイトル（Claude Codeのセッション名）からキーを取る。続けて同じリポジトリの `git worktree list` からキーを含むブランチのworktreeを探し、見つからなければローカルブランチを探す

PRは `gh pr list --head <ブランチ>` で探し、見つからなければタイトルにキーを含むPRを探す。複数あればOPENを優先する。PRのheadブランチが手順で決めたブランチと違うときは、PR番号の後ろに `⚠` を付ける。

キーは `ABC-12` が `ABC-123` のブランチに一致しないよう、数字の途中で切れた一致を除外する。

## トークン

herdrのサイドバーは、トークンごとに固定の色しか指定できない。そこでプラグインは状態ごとに別のトークンを用意し、該当する1つにだけ値を入れる。値が空のトークンは表示されない。

| 種類 | トークン | アイコン（Nerd Font） | 値が入る条件 |
|---|---|---|---|
| Jira | `$jira_new` / `$jira_doing` / `$jira_done` | `nf-dev-jira` | ステータスカテゴリが To Do / 進行中 / 完了。値はキーとステータス名 |
| Jira | `$jira_err` | `nf-dev-jira` | 認証失敗（`auth✗`）、課題が見つからない（`not found`） |
| PR | `$pr_open` / `$pr_draft` / `$pr_merged` / `$pr_closed` | `nf-oct-git_pull_request` / `_draft` / `git_merge` / `_closed` | PRの状態。値は `#番号` |
| CI | `$ci_ok` / `$ci_ng` / `$ci_wait` | `nf-oct-check` / `x` / `dot_fill` | OPENのPRのチェック結果 |
| レビュー | `$rv_ok` / `$rv_ng` / `$rv_wait` | `nf-oct-eye` | OPENのPRの承認 / 変更要求 / レビュー待ち |
| worktree | `$wt` | `nf-oct-git_branch` | worktreeのディレクトリ名。キーは `$jira_*` に出ているので取り除く。取り除くと空になるときはディレクトリ名をそのまま出す |

Jiraを取得できないときは、`$jira_new` にキーだけを出す。

## board

アクション `jira-pr: open board` は、エージェントpaneとworktreeの一覧をポップアップのfzfで開く。

一覧の上の段にはエージェントpaneを並べる。その下には、エージェントpaneがあるリポジトリのworktreeのうち、どのpaneにも紐づかずブランチ名にキーを含むものを並べる。PRがmergedのまま残ったworktreeも、ここで見つけられる。各行は、場所、Jiraキー、PRとCI・レビュー、worktree、Jiraのステータスの順に表示する。色はサイドバーと同じ規則で付ける。

| キー | 動作 |
|---|---|
| `enter` / ダブルクリック | PRをブラウザで開く |
| `ctrl-j` | Jiraの課題をブラウザで開く |
| `ctrl-o` | worktreeをエディタで開く。エディタは `config.env` の `EDITOR_CMD`（既定は `zed`）で決める |
| `ctrl-f` | そのpaneへ移動する。paneの無いworktreeの行では、worktreeをherdrのworkspaceとして開く |
| `ctrl-r` | キャッシュを使わずに取り直す |
| `esc` | 閉じる |

paneの無いworktreeのPRはブランチ名だけで探し、タイトル検索はしない。worktreeごとに `gh` を6並列で呼ぶので、worktreeが30個あるリポジトリで初回は8秒ほどかかる。2回目以降は `CACHE_TTL_SEC` の間キャッシュを使う。

boardの色はスクリプトに書いてある。`config.toml` のサイドバーの色を変えても、boardの色は変わらない。

## 更新のタイミング

| きっかけ | 対象 |
|---|---|
| herdrサーバの起動 | 全エージェントpane。トークンはサーバの再起動で消えるので報告し直す |
| `worktree.created` / `opened` / `removed` | 全エージェントpane |
| `pane.agent_detected` / `focused` | そのpane |
| アクション `jira-pr: refresh all panes` | 全エージェントpane。キャッシュを使わず取り直す |

herdr 0.9.5では `pane.updated` をプラグインのフックに使えない。このため、起動後にセッション名が付いたpaneは、次にフォーカスしたときに表示される。

JiraとPRの取得結果は `CACHE_TTL_SEC`（既定300秒）の間キャッシュする。ghやJiraへの問い合わせが失敗したときは、期限切れのキャッシュを表示する。

## 設定

### 1. プラグインを登録する

```bash
herdr plugin link ~/Developer/github/herdr-plugins/jira-pr
```

### 2. Jiraの接続先を書く

```bash
CONFIG_DIR="$(herdr plugin config-dir ymegane.jira-pr)"
cp ~/Developer/github/herdr-plugins/jira-pr/config.example.env "${CONFIG_DIR}/config.env"
```

`config.env` の `JIRA_EMAIL` をAtlassianアカウントのメールアドレスに書き換える。

### 3. JiraのAPIトークンをKeychainに入れる

<https://id.atlassian.com/manage-profile/security/api-tokens> の「Create API token」でAPIトークンを発行する。スコープ付きトークン（Create API token with scopes）はサイトURLへの直接アクセスで401になるので使わない。

トークンをクリップボードにコピーし、次のコマンドで登録する。

```bash
printf 'add-generic-password -U -s herdr-jira-pr -a <JIRA_EMAIL と同じ値> -w "%s"\n' "$(pbpaste)" | security -i
```

`security add-generic-password ... -w` の対話入力は128文字で切り詰められ、190文字前後あるAPIトークンが壊れる。このため対話入力ではなく、`security -i` の標準入力からコマンドとして渡す。`printf` はシェルの組み込みコマンドなので、トークンは `ps` に出ない。

登録できたかは、次のコマンドが190前後を返すことで確かめる。

```bash
security find-generic-password -s herdr-jira-pr -a <JIRA_EMAIL と同じ値> -w | tr -d '\n' | wc -c
```

Jiraを設定しなくても、Jiraの行にキーだけを表示し、PRとworktreeの行は動く。

### 4. サイドバーにトークンを出す

`~/.config/herdr/config.toml` のエージェント行にトークンを足し、状態ごとに色を付ける。色はCatppuccin Mochaに合わせた例。

```toml
[ui.sidebar.agents.rows_by_agent]
claude = [
  ["state_icon", "machine", "workspace", "tab"],
  ["terminal_title_stripped"],
  ["agent"],
  [
    { token = "$jira_new", fg = "#9399b2" },
    { token = "$jira_doing", fg = "#89b4fa" },
    { token = "$jira_done", fg = "#a6e3a1" },
    { token = "$jira_err", fg = "#f38ba8" },
  ],
  [
    { token = "$pr_open", fg = "#a6e3a1" },
    { token = "$pr_draft", fg = "#9399b2" },
    { token = "$pr_merged", fg = "#cba6f7" },
    { token = "$pr_closed", fg = "#f38ba8" },
    { token = "$ci_ok", fg = "#a6e3a1" },
    { token = "$ci_ng", fg = "#f38ba8" },
    { token = "$ci_wait", fg = "#f9e2af" },
    { token = "$rv_ok", fg = "#a6e3a1" },
    { token = "$rv_ng", fg = "#f38ba8" },
    { token = "$rv_wait", fg = "#f9e2af" },
  ],
  [{ token = "$wt", dim = true }],
]
```

同じ行に並べたトークンのあいだには、herdrが ` · ` を挟む。サイドバーの既定幅は26桁なので、PRとworktreeを同じ行にすると途中で切れる。

反映には `herdr server reload-config` を実行する。

### 5. キーバインド（任意）

```toml
[[keys.command]]
key = "prefix+i"
type = "plugin_action"
command = "ymegane.jira-pr.board"
```

`prefix+o`（通知先へ移動）や `prefix+j`（下のpaneへ移動）はherdrの既定で使われている。空いているキーは `herdr --default-config` で確かめる。

アクションは `board`、`refresh`、`open-pr`、`open-jira`、`open-editor` の5つ。`open-*` はフォーカス中のpaneを対象にする。紐づくPR、Jira課題、worktreeが無いときは、herdrのトースト通知で知らせる。通知の出し方は `[ui.toast]` の `delivery` に従う。

`EDITOR_CMD` はherdrサーバの `PATH` で探される。`code` や `zed` のCLIが見つからないときは、`"open -a Cursor"` のように `open -a` で書くかフルパスで書く。

## 動作確認

`status` サブコマンドは、全エージェントpaneの解決結果を表で出す。トークンは報告しない。

```bash
~/Developer/github/herdr-plugins/jira-pr/bin/jira-pr status | column -t -s "$(printf '\t')"
```

フックの実行ログは `herdr plugin log list --plugin ymegane.jira-pr` で見られる。

## 依存

`bash`、`jq`、`git`、`gh`（認証済み）、`curl`、`security`。boardには `fzf` も要る。アイコンの表示には、herdrを動かす端末のフォントがNerd Fontである必要がある。
