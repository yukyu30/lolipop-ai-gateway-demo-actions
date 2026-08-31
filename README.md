# Lolipop AI Gateway Demo Actions

Lolipop AI Gateway と Claude Code Action を試すためのデモリポジトリです。

## セットアップ

Claude Code Action のセットアップについては、公式ガイドを参照してください。

- [Claude Code Action Setup Guide](https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md)

## Claude Code Review

Pull Requestを作成すると、Claude Code Actionによるコードレビューが実行されます。Claude APIへのリクエストは、ロリポップ！AI Gatewayを経由します。

ガードレールの動作確認には、ダミーのメールアドレス `ugo@example.com` を使用します。

レビューworkflowは、Claude Code Action公式のcomprehensive PR review例をもとに、Haikuモデルで軽量に実行します。

Claude Codeのthinkingを有効にしたまま、PR情報を取得するツール往復を実行します。

## 検証メモ

- [Claude Code Actionをthinking有効で使う設定](docs/claude-code-action-thinking-setup.md)
- [Claude Code Action × Lolipop AI Gateway: tool利用検証メモ](docs/claude-code-action-tool-use.md)
