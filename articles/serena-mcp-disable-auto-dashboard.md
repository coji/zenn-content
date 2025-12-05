---
title: "Serena MCP でブラウザが勝手に開くのを止める - 設定ファイル1行で解決"
emoji: "🛠"
type: "tech"
topics: ["mcp", "serena", "claudecode"]
published: true
---

## これはなに？

Claude Code で Serena MCP を使っていると、起動するたびにブラウザでダッシュボードが開いてしまいます。複数プロジェクトで有効にしていると、Claude Code を動かすたびにタブが何個も開いてつらい状況になります。設定ファイル1行で解決できたので、その方法を共有します。

## Claude Code にコピペで依頼する

以下のプロンプトを Claude Code に貼り付ければ、設定を自動で行ってくれます。

```text
~/.serena/serena_config.yml を開いて、
web_dashboard_open_on_launch: false を追加して。
ファイルがなければ作成して。
```

これだけで完了です。以下は手動で設定したい場合や、何が起きているか知りたい場合の説明です。

## なぜブラウザが何度も開くのか

Serena はデフォルトで Web ダッシュボード機能が有効になっています。`web_dashboard_open_on_launch` オプションがデフォルトで有効なため、MCP サーバーが起動するたびにブラウザが自動で開きます。

Claude Code では MCP サーバーが頻繁に再起動するため、セッション中にブラウザのタブがどんどん増えていきます。ユーザグローバルで Serena を入れて複数プロジェクトで有効にしていると、特に顕著です。

## 手動で設定する場合

設定ファイルのパスは `~/.serena/serena_config.yml` です。

```bash
mkdir -p ~/.serena
```

設定ファイルに以下を追加します。

```yaml:~/.serena/serena_config.yml
web_dashboard_open_on_launch: false
```

これで次回起動時からブラウザは開かなくなります。ダッシュボード自体は無効にならないので、必要なときは http://localhost:24282/dashboard/index.html に手動でアクセスできます。

## まとめ

設定ファイルに1行追加するだけで、タブが増え続ける問題は解消されました。

## 別の方法

この記事を公開したあと、起動オプションで `--enable-web-dashboard` を `false` にする方法を教えていただきました。MCP サーバーの設定で起動引数を指定できる場合は、こちらの方法でも解決できます。

https://x.com/ushiro_noko/status/1996537621238128716

## 参考

- [The Dashboard and GUI Tool — Serena Documentation](https://oraios.github.io/serena/02-usage/060_dashboard.html)
- [oraios/serena - GitHub](https://github.com/oraios/serena)
