# Claude Code Action × Lolipop AI Gateway: tool利用検証メモ

初回検証日: 2026-08-27（JST）

thinking対応の再検証日: 2026-08-31（JST）

## 結論

Lolipop AI Gatewayは、Claude Code Actionから利用できる。通常の1ターン応答に加え、thinkingを有効にした`tool_use`／`tool_result`の複数ターンと、PRレビューコメントの投稿まで完走する。

Claude Code Action公式の[comprehensive PR review例](https://github.com/anthropics/claude-code-action/blob/main/examples/pr-review-comprehensive.yml)をHaiku向けに軽量化したworkflowでは、Claudeが`gh pr view`と`gh pr diff`を実行し、PRへレビューコメントを投稿するところまで成功した。

一方、`/install-github-app`が生成する公式`code-review`プラグインは、小規模なPRでも多数のsubagentとtoolを起動する。今回の環境では権限拒否が繰り返され、29ターン・`$1.81849155`利用後にHTTP 402で終了した。この挙動はClaude Codeの[issue #26227](https://github.com/anthropics/claude-code/issues/26227)でも報告されているため、軽量なデモでは直接プロンプト方式を採用した。

2026-08-27の初回検証では、tool実行後のMessages APIリクエストで、直前の`thinking`と`tool_use`、新しい`tool_result`を会話履歴として再送するとHTTP 400 `Unsupported Anthropic content block`が返された。

2026-08-31にthinking無効化設定を削除して再検証したところ、`thinking` blockを含む6ターンが成功した。初回に確認したthinking blockの非互換は、AIゲートウェイ側の修正によって解消したと判断できる。

## 検証環境

- GitHub Action: `anthropics/claude-code-action@v1`
- Claude Code: `2.1.246`
- Base URL: `https://ai-gateway.lolipop.jp/anthropic`
- 認証: GitHub Actions Secret `AI_GATEWAY_API_KEY`を`anthropic_api_key`へ指定
- モデル:
  - `anthropic/claude-haiku-4-5`
  - `anthropic/claude-sonnet-4-6`
- 診断workflow: [diagnose-ai-gateway.yml](../.github/workflows/diagnose-ai-gateway.yml)

## 証拠一覧

| 検証 | 結果 | GitHub Actions |
| --- | --- | --- |
| 最小Messages API | HTTP 200、本文`OK` | [run 32992505333](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32992505333) |
| Claude Code Action 1ターン | `num_turns: 1`、`result: "OK"` | [run 32992505333](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32992505333) |
| デフォルトthinkingでBash tool往復 | tool実行後の2ターン目でHTTP 400 | [run 32993298155](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32993298155) |
| 直接HTTP、thinkingなし | 1ターン目200、2ターン目200 | [run 32995799540](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32995799540) |
| 直接HTTP、thinkingあり | 1ターン目200、2ターン目400 | [run 32995799540](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32995799540) |
| Claude Code Action、thinking無効、Bash tool往復 | `num_turns: 2`、`result: "TOOL_DONE"` | [run 32995941818](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32995941818) |
| 実際の`code-review`プラグイン、デフォルトthinking | `num_turns: 2`、`is_error: true` | [run 32991302884](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32991302884) |
| 実際の`code-review`プラグイン、thinking無効 | 12ターン進行後、HTTP 402残高不足で終了 | [run 32996846956](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32996846956) |
| `code-review`、最大出力4,096・並列数2 | 29ターン、`$1.81849155`利用後にHTTP 402 | [run 32997551686](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32997551686) |
| Haiku comprehensive review、`max-turns: 6` | レビュー本文は生成したが、必要な7ターンに対して上限不足 | [run 32999160681](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32999160681) |
| Haiku comprehensive review、`max-turns: 10` | `gh pr view`、`gh pr diff`を実行し、6ターンでレビュー投稿に成功 | [run 32999774810](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32999774810) |
| PRレビューコメント | 指摘なし、ApprovedとしてPR #20へ投稿 | [issuecomment-5429431662](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/pull/20#issuecomment-5429431662) |
| thinking有効のHaiku comprehensive review | thinking 1件、tool result 5件、6ターンでレビュー投稿に成功 | [run 33366681419](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/33366681419) |
| thinking有効時のPRレビューコメント | 指摘なしとしてPR #21へ投稿 | [issuecomment-5474914697](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/pull/21#issuecomment-5474914697) |

## 対照実験の詳細

### 1. thinkingなしのtool往復

[run 32995799540](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32995799540)では、Anthropic互換Messages APIへ直接リクエストした。

1回目はtool定義とtool利用を求めるuser messageを送信した。

```text
Direct tool first-turn HTTP status: 200
{"type":"message","stop_reason":"tool_use","content_types":["tool_use"],"tool_use":{"name":"echo_value","input":{"value":"TOOL_OK"}}}
```

返された`tool_use`をassistant messageとして入れ、対応する`tool_result`をuser messageとして送信した。

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_...",
      "content": "TOOL_OK"
    }
  ]
}
```

2回目も成功した。

```text
Direct tool second-turn HTTP status: 200
{"type":"message","stop_reason":"end_turn","content_types":["text"]}
```

この結果から、AIゲートウェイが`tool_use`と`tool_result`を処理できることを確認した。

### 2. thinkingありのtool往復

同じrunで、1回目のリクエストにAnthropicのthinkingを有効化した。

```text
Thinking tool first-turn HTTP status: 200
{"type":"message","stop_reason":"tool_use","content_types":["thinking","tool_use"],"tool_use":{"name":"echo_value","input":{"value":"TOOL_OK"}}}
```

返された署名付き`thinking`と`tool_use`を変更せずassistant messageとして再送し、対応する`tool_result`を追加した。2回目だけが失敗した。

```text
Thinking tool second-turn HTTP status: 400
{"error_type":"bad_request","error_message":"Unsupported Anthropic content block"}
```

thinking以外の条件が同じ対照実験なので、400の原因は会話履歴中のAnthropic `thinking` blockと切り分けられる。

### 3. Claude Code Actionでthinkingを無効化

[run 32995941818](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32995941818)では、Claude Codeの公式設定でthinkingを無効化した。

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.AI_GATEWAY_API_KEY }}
    settings: |
      {
        "env": {
          "CLAUDE_CODE_DISABLE_THINKING": "1",
          "MAX_THINKING_TOKENS": "0"
        }
      }
    prompt: |
      Use the Bash tool exactly once to run: printf TOOL_OK
      After reading the tool result, reply with exactly TOOL_DONE.
    claude_args: >-
      --model anthropic/claude-haiku-4-5
      --max-turns 3
      --allowedTools "Bash(printf TOOL_OK)"
```

Actionログには、次の順序が残った。

1. Claudeが`Bash`の`tool_use`を返す
2. Claude Codeが`printf TOOL_OK`を実行する
3. `tool_result`の本文が`TOOL_OK`になる
4. AIゲートウェイへの2回目のリクエストが成功する
5. 最終応答が`TOOL_DONE`になる

```text
"type": "tool_use"
"command": "printf TOOL_OK"

"type": "tool_result"
"content": "TOOL_OK"

"num_turns": 2
"subtype": "success"
"result": "TOOL_DONE"
```

## Claude Code Reviewの設定

[claude-code-review.yml](../.github/workflows/claude-code-review.yml)では、公式comprehensive PR review例の観点を保ちつつ、Haikuが必要な情報だけを取得する直接プロンプト方式にしている。

```yaml
env:
  ANTHROPIC_BASE_URL: https://ai-gateway.lolipop.jp/anthropic

steps:
  - uses: actions/checkout@v6

  - id: claude-review
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

        Perform a concise but comprehensive code review covering code quality,
        security, performance, tests, and documentation accuracy.
        Report only high-confidence, actionable findings.
      claude_args: >-
        --model anthropic/claude-haiku-4-5
        --max-turns 15
        --allowedTools "mcp__github_inline_comment__create_inline_comment,Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*)"
```

現在のworkflowは`CLAUDE_CODE_DISABLE_THINKING`も`MAX_THINKING_TOKENS`も設定せず、Claude Codeのthinkingを有効にしている。`--max-turns 15`は、thinking有効時に最初の試行が12ターンとなり、従来の上限10を超えたため余裕を持たせた値である。

### 公式`code-review`プラグインで完走しなかった理由

[run 32996846956](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32996846956)では、thinkingを無効化した別のworkflowで公式`code-review`プラグインを実行した。Actionは`anthropic/claude-sonnet-4-6`を使用して12ターン進行し、モデル利用料金も記録された。

```text
"num_turns": 12
"total_cost_usd": 0.34307594999999996
"result": "API Error: 402 Insufficient balance"
```

このrunではHTTP 400 `Unsupported Anthropic content block`は発生していない。したがってthinking無効化の回避策は実際のプラグインでも有効と判断できる。ただし402で停止したため、プラグインによるレビュー投稿は完走しなかった。

AIゲートウェイの[リクエストの上限](https://ai-gateway.lolipop.jp/docs/api-reference/limits)によると、クレジットは応答後の実利用量だけで判定されるのではない。リクエスト受付時に`max_tokens`を上限とする料金を予約し、応答後に実利用量で精算する。`code-review`プラグインは複数のsubagentを並列起動するため、各リクエストの予約が同時に積み上がる。

失敗runではClaude Codeがモデルの最大出力を32,000 tokenとして扱っていた。画面上にクレジットが残っていても、この予約額の合計が一時的に残高を超えると402になり得る。対策として、[Claude Code公式の環境変数](https://code.claude.com/docs/en/env-vars)を使い、1リクエストの最大出力と並列数を抑えている。

```json
{
  "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "4096",
  "CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY": "2"
}
```

[run 32997551686](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32997551686)でこの設定を検証したところ、前回の12ターンから29ターンまで進行したが、レビュー完了前に再び402になった。

```text
"num_turns": 29
"total_cost_usd": 1.81849155
"result": "API Error: 402 Insufficient balance"
```

公式`code-review`プラグインは複数の専門subagentを使うため、小規模なデモでも通常の1ターン実行より大幅に高コストになる。約493円の表示残高がある状態でも、実利用額の累積と次リクエストの予約額により完走できなかった。また、許可されていないtoolをsubagentが繰り返し要求し、17件のpermission denialが発生した。低残高でレビューまで確認する用途には、次のHaiku comprehensive reviewの方が適している。

### Haiku comprehensive reviewの完走結果

最初は`--max-turns 6`に設定したため、レビュー結果自体は生成されたものの、実際に必要だった7ターンに対して上限が不足し、Actionが失敗扱いになった。[run 32999160681](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32999160681)を受けて上限を10へ変更した。

[run 32999774810](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32999774810)ではActionが52秒で成功し、サニタイズ済みの実行証跡に次のtool利用が記録された。

```json
{
  "tool_uses": [
    {"name": "Read", "command": ""},
    {"name": "Bash", "command": "gh pr view 20 --json title,body,state,commits,files"},
    {"name": "Bash", "command": "gh pr diff 20"},
    {"name": "mcp__github_comment__update_claude_comment", "command": ""}
  ],
  "result": {
    "subtype": "success",
    "is_error": false,
    "num_turns": 6,
    "total_cost_usd": 0.0497607
  }
}
```

Claudeは取得したPR情報と差分をレビューし、[PR #20のコメント](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/pull/20#issuecomment-5429431662)へコード品質、セキュリティ、性能、テスト、ドキュメントの各観点を記載した。今回はドキュメント1行の変更で問題がなかったため、最終判定は`Approved - Ready to merge`だった。

これにより、以下をend-to-endで確認できた。

1. Claude Code Actionがロリポップ！AIゲートウェイ経由で起動する
2. ClaudeがBash toolでPR情報と差分を取得する
3. `tool_result`を受けた後も複数ターンを継続する
4. Claude Code ActionがGitHubのPRへレビューコメントを投稿する
5. GitHub Actions jobが成功終了する

### thinking有効での再検証結果

2026-08-31にthinking無効化設定を削除し、[PR #21](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/pull/21)で再検証した。

最初の[run 33366429216](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/33366429216)は、Claudeの処理自体は`is_error: false`で完了していたが、12ターンに対して`--max-turns 10`だったためActionが失敗扱いになった。HTTP 400は発生していない。上限を15へ変更した[run 33366681419](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/33366681419)は54秒で成功した。

サニタイズ済みログでは、thinking本文を公開せず、content blockの種類と件数だけを記録した。

```json
{
  "assistant_content_types": ["text", "thinking", "tool_use"],
  "thinking_block_count": 1,
  "tool_result_count": 5,
  "tool_uses": [
    {"name": "Bash", "command": "gh pr view 21"},
    {"name": "Bash", "command": "gh pr diff 21"},
    {"name": "mcp__github_comment__update_claude_comment", "command": ""}
  ],
  "result": {
    "subtype": "success",
    "is_error": false,
    "num_turns": 6,
    "total_cost_usd": 0.048063299999999996
  }
}
```

Action本体の結果には`permission_denials_count: 0`も記録された。Claudeは`gh pr view`と`gh pr diff`でPRを調査し、[PR #21のレビューコメント](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/pull/21#issuecomment-5474914697)を投稿した。

この結果から、2026-08-31時点のAIゲートウェイは、Claude Codeがtool利用後に再送する署名付き`thinking` content blockを受理できることを確認した。

## thinking block対応の経緯

2026-08-27時点では、回避設定なしでClaude Codeを使うために、tool利用後のMessages APIリクエストで署名付き`thinking` content blockを受理する対応が必要だった。

当時から`tool_result`自体はthinkingなしの対照実験でHTTP 200になっていた。2026-08-31の再検証ではthinkingを含む往復も成功したため、現在のworkflowではthinking無効化の回避設定を削除している。

## 参考資料

- [Lolipop AI Gateway: Anthropic SDK](https://ai-gateway.lolipop.jp/docs/guides/sdks/anthropic-sdk)
- [Claude Code: Environment variables](https://code.claude.com/docs/en/env-vars)
- [Anthropic: Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)
- [Claude Code Action: Comprehensive PR Review Example](https://github.com/anthropics/claude-code-action/blob/main/examples/pr-review-comprehensive.yml)
- [Claude Code: Code Review Plugin](https://github.com/anthropics/claude-code/tree/main/plugins/code-review)
- [Claude Code issue #26227](https://github.com/anthropics/claude-code/issues/26227)
