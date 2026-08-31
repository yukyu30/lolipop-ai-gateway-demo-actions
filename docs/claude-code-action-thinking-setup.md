# Claude Code Actionをthinking有効で使う設定

ロリポップ！AIゲートウェイ経由のClaude Code Actionで、thinkingを有効にしたままPRレビューとtool利用を行うための設定をまとめる。

2026年8月31日の検証では、`thinking`、`tool_use`、`tool_result`を含む複数ターンと、GitHubへのレビューコメント投稿まで成功した。

## 前提

- GitHub AppとClaude Code Actionの初期設定が完了している
- Repository Actions secretに`AI_GATEWAY_API_KEY`を登録している
- AIゲートウェイで利用可能なモデルを指定する
- PRの内容が外部のAIゲートウェイへ送信されることを許容できる

この構成では、`AI_GATEWAY_API_KEY`をActionの`anthropic_api_key`へ渡す。`ANTHROPIC_AUTH_TOKEN`は使用しない。

## 1. thinking無効化設定を削除する

以前の回避策として次の環境変数を設定している場合は、両方とも削除する。

```json
{
  "CLAUDE_CODE_DISABLE_THINKING": "1",
  "MAX_THINKING_TOKENS": "0"
}
```

現在の設定では、出力量の上限だけを残している。

```yaml
settings: |
  {
    "env": {
      "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "2048"
    }
  }
```

## 2. AIゲートウェイと認証情報を設定する

jobの`env`へAnthropic互換APIのbase URLを設定し、secretをActionへ渡す。

```yaml
env:
  ANTHROPIC_BASE_URL: https://ai-gateway.lolipop.jp/anthropic

steps:
  - uses: anthropics/claude-code-action@v1
    with:
      anthropic_api_key: ${{ secrets.AI_GATEWAY_API_KEY }}
```

APIキーをworkflowファイルへ直接記載しない。

## 3. toolとターン上限を明示する

このリポジトリではHaikuを指定し、PRの取得とコメントに必要なtoolだけを許可している。

```yaml
claude_args: >-
  --model anthropic/claude-haiku-4-5
  --max-turns 15
  --allowedTools "mcp__github_inline_comment__create_inline_comment,Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*)"
```

thinkingを有効にすると、同じレビューでも必要なターン数が変動する。今回の最初の試行は12ターンだったため、上限を15にした。

## 4. 完成形のworkflow

主要部分は次のとおり。実際に使用している完全なファイルは[claude-code-review.yml](../.github/workflows/claude-code-review.yml)を参照する。

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  claude-review:
    if: >-
      github.event.pull_request.head.repo.full_name == github.repository &&
      github.event.pull_request.user.login == 'yukyu30'
    runs-on: ubuntu-latest
    env:
      ANTHROPIC_BASE_URL: https://ai-gateway.lolipop.jp/anthropic
    permissions:
      contents: read
      pull-requests: write
      id-token: write

    steps:
      - uses: actions/checkout@v6

      - name: Run comprehensive PR review
        id: claude-review
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.AI_GATEWAY_API_KEY }}
          settings: |
            {
              "env": {
                "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "2048"
              }
            }
          track_progress: true
          prompt: |
            REPO: ${{ github.repository }}
            PR NUMBER: ${{ github.event.pull_request.number }}

            Use the allowed `gh pr view` and `gh pr diff` Bash tools to inspect
            this pull request before reviewing it.

            Perform a concise but comprehensive code review covering code
            quality, security, performance, tests, and documentation accuracy.
            Report only high-confidence, actionable findings.
          claude_args: >-
            --model anthropic/claude-haiku-4-5
            --max-turns 15
            --allowedTools "mcp__github_inline_comment__create_inline_comment,Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*)"
```

`if`の条件は、このデモで信頼できる同一リポジトリのowner作成PRだけを対象にするためのガードである。別のownerで利用する場合はGitHubのloginを変更する。

## 5. thinkingとtool往復を確認する

このリポジトリのworkflowは、`execution_file`から次の項目をサニタイズしてActionsログへ出す。

- assistant responseに含まれるcontent blockの種類
- `thinking` blockの件数
- `tool_result`の件数
- tool名と実行コマンド
- 最終結果、ターン数、費用

thinking本文や署名は出力しない。成功時は次のような証跡になる。

```json
{
  "assistant_content_types": ["text", "thinking", "tool_use"],
  "thinking_block_count": 1,
  "tool_result_count": 5,
  "tool_uses": [
    {"name": "Bash", "command": "gh pr view 21"},
    {"name": "Bash", "command": "gh pr diff 21"}
  ],
  "result": {
    "subtype": "success",
    "is_error": false,
    "num_turns": 6
  }
}
```

実測結果は[run 33366681419](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/33366681419)と[PR #21のレビューコメント](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/pull/21#issuecomment-5474914697)で確認できる。

## workflow変更時の注意

Claude Code Actionは、PRで実行するworkflowがdefault branch上のファイルと同一か検証する。workflowを初めて追加したPRや、workflow自体を変更するPRでは、次のメッセージで実行がskipされることがある。

```text
The workflow file must exist and have identical content to the version on the repository's default branch.
```

この場合はworkflowをdefault branchへ反映してから、別のPRで動作を検証する。

## トラブルシューティング

### 成功結果なのにturn上限で失敗する

```text
Claude reported a successful result after 12 turns, exceeding the configured maximum of 10
```

`is_error: false`ならAPIやthinkingの失敗ではない。実際の`num_turns`より大きい`--max-turns`へ変更する。

### `Unsupported Anthropic content block`になる

tool実行後のリクエストでHTTP 400になる場合は、利用中のAIゲートウェイ環境へthinking対応が反映されているか確認する。一時回避だけが必要なら`CLAUDE_CODE_DISABLE_THINKING=1`と`MAX_THINKING_TOKENS=0`でthinkingを無効化できる。

### HTTP 402になる

残高不足または`max_tokens`を基準にした一時的な料金予約が原因になり得る。安価なモデル、少ない最大出力、必要最小限のtool、直接プロンプト方式を利用する。

## 結果

2026年8月31日時点では、ロリポップ！AIゲートウェイ経由のClaude Code Actionでthinkingを無効化する必要はない。明示したtoolを使ってPRを調査し、`tool_result`を返して処理を継続し、レビューコメントを投稿できる。
