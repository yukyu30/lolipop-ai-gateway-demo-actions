# Claude Code Action × Lolipop AI Gateway: tool利用検証メモ

検証日: 2026-08-27（JST）

## 結論

Lolipop AI Gatewayは、Claude Code Actionから利用できる。通常の1ターン応答に加え、`tool_use`から`tool_result`を返す複数ターンも、Claude Codeのthinkingを無効化すれば完走する。

公式`code-review`プラグインもthinking無効化後は12ターンまで進行した。検証runは最終的にHTTP 402 `Insufficient balance`で終了したためレビュー完走までは確認できていないが、最新の停止原因はcontent blockの非互換ではない。

デフォルト設定のClaude Codeはthinking blockを生成する。tool実行後の次のMessages APIリクエストで、直前の`thinking`と`tool_use`、新しい`tool_result`を会話履歴として再送すると、2026-08-27時点のAIゲートウェイはHTTP 400 `Unsupported Anthropic content block`を返す。

比較実験では、thinkingを含めなければ同じ`tool_use`／`tool_result`往復がHTTP 200で成功した。したがって、未対応なのは`tool_result`ではなく、tool利用時に再送されるAnthropic `thinking` content blockと判断できる。

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

[claude-code-review.yml](../.github/workflows/claude-code-review.yml)では、公式`code-review`プラグインを利用しながらthinkingを無効化している。

```yaml
env:
  ANTHROPIC_BASE_URL: https://ai-gateway.lolipop.jp/anthropic

steps:
  - uses: anthropics/claude-code-action@v1
    with:
      anthropic_api_key: ${{ secrets.AI_GATEWAY_API_KEY }}
      settings: |
        {
          "env": {
            "CLAUDE_CODE_DISABLE_THINKING": "1",
            "MAX_THINKING_TOKENS": "0",
            "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "4096",
            "CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY": "2"
          }
        }
      plugin_marketplaces: "https://github.com/anthropics/claude-code.git"
      plugins: "code-review@claude-code-plugins"
      prompt: "/code-review:code-review ${{ github.repository }}/pull/${{ github.event.pull_request.number }}"
      claude_args: "--model anthropic/claude-sonnet-4-6"
```

`CLAUDE_CODE_DISABLE_THINKING=1`はthinkingを強制的に無効化するClaude Code公式の環境変数である。`MAX_THINKING_TOKENS=0`もthinkingを無効化する設定として併記している。

### `code-review`プラグインの実行結果

[run 32996846956](https://github.com/yukyu30/lolipop-ai-gateway-demo-actions/actions/runs/32996846956)では、上記設定で公式`code-review`プラグインを実行した。Actionは`anthropic/claude-sonnet-4-6`を使用して12ターン進行し、モデル利用料金も記録された。

```text
"num_turns": 12
"total_cost_usd": 0.34307594999999996
"result": "API Error: 402 Insufficient balance"
```

このrunではHTTP 400 `Unsupported Anthropic content block`は発生していない。したがってthinking無効化の回避策は実際のプラグインでも有効と判断できる。ただし402で停止したため、レビュー投稿までのend-to-end完走は未確認である。

AIゲートウェイの[リクエストの上限](https://ai-gateway.lolipop.jp/docs/api-reference/limits)によると、クレジットは応答後の実利用量だけで判定されるのではない。リクエスト受付時に`max_tokens`を上限とする料金を予約し、応答後に実利用量で精算する。`code-review`プラグインは複数のsubagentを並列起動するため、各リクエストの予約が同時に積み上がる。

失敗runではClaude Codeがモデルの最大出力を32,000 tokenとして扱っていた。画面上にクレジットが残っていても、この予約額の合計が一時的に残高を超えると402になり得る。対策として、[Claude Code公式の環境変数](https://code.claude.com/docs/en/env-vars)を使い、1リクエストの最大出力と並列数を抑えている。

```json
{
  "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "4096",
  "CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY": "2"
}
```

## AIゲートウェイ側で対応が望まれる内容

回避設定なしでClaude Codeを使うには、tool利用後のMessages APIリクエストで、直前のassistant messageに含まれる署名付き`thinking` content blockを受理し、Anthropic APIへ透過または適切に変換する必要がある。

`tool_result`の追加対応が必要なのではない。thinkingなしの対照実験で、`tool_result`を含む2回目のリクエストが既にHTTP 200になっているためである。

## 参考資料

- [Lolipop AI Gateway: Anthropic SDK](https://ai-gateway.lolipop.jp/docs/guides/sdks/anthropic-sdk)
- [Claude Code: Environment variables](https://code.claude.com/docs/en/env-vars)
- [Anthropic: Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)
