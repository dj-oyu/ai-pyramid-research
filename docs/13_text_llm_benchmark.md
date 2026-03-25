# テキスト LLM ベンチマーク: Daily Summary 生成

## 目的

pet-album の1日分サマリー生成を高速化できるか検証。
Qwen3-VL-2B (VLM, テキストモード) vs Qwen3-1.7B (テキスト専用 LLM) を比較。

## テスト条件

- デバイス: M5Stack AI Pyramid Pro (AX650N_M5stack_8G)
- 入力: 2026-03-24 の valid キャプション 50件
- プロンプト: "Summarize this cat's day..." (2-3文, plain text)
- max_tokens: 256, temperature: 0.3
- 各条件3回実行の平均

## 結果

### 英語出力

| モデル | 条件 | 平均レスポンス | 品質 |
|--------|------|--------------|------|
| **Qwen3-VL-2B** | text-only (画像なし) | **13.5秒** | クリーン、簡潔、指示追従◎ |
| Qwen3-1.7B | thinking ON (default) | 38.6秒 | `<think>` で内部推論出力、遅い |
| Qwen3-1.7B | `/no_think` system prompt | 20.8秒 | 空 `<think></think>` タグ残留、内容は具体的 |

### 日本語出力

| モデル | 条件 | 平均レスポンス | 品質 |
|--------|------|--------------|------|
| **Qwen3-VL-2B** | text-only | **14.0秒** | 自然な日本語、フォーマットクリーン |
| Qwen3-1.7B | `/no_think` | 19.8秒 | 「bowl」未翻訳混入、think タグ残留 |

## 分析

### Qwen3-1.7B の thinking モード

- Qwen3 ファミリーはデフォルトで Chain-of-Thought (thinking) が有効
- system メッセージに `/no_think` を入れると空の `<think></think>` を出力して本文に進む
- `chat_template_kwargs: {enable_thinking: false}` は axllm では無視される
- thinking OFF でも VL-2B より 1.5 倍遅い

### 速度差の原因

- VL-2B のコンテキスト長: 3,584 tokens (C256-P3584-CTX4095)
- 1.7B のモデルサイズ: 2.7GB vs VL-2B: 3.1GB
- VL-2B は vision encoder 分のオーバーヘッドがあるはずだが、テキストのみだと高速
- 1.7B は thinking 無効でも内部の推論パスが異なる可能性

### 出力品質

| 項目 | VL-2B | 1.7B /no_think |
|------|-------|----------------|
| 指示追従 (文数制御) | ◎ | ○ やや多い |
| フォーマット制御 | ◎ クリーン | △ think タグ残留要除去 |
| 内容の具体性 | ○ | ◎ |
| 日本語品質 | ○ 自然 | △ 英単語混入 |

## 結論

**Daily summary には Qwen3-VL-2B をテキストモードで使うのがベスト。**

- 速度: 1.5倍速い (14秒 vs 20秒)
- フォーマット: 後処理不要
- 日本語: より自然
- テキスト専用 LLM への切り替えは不要

Qwen3-1.7B は内容の具体性では優れるが、速度・フォーマット・日本語品質で劣る。
モデル切り替え (systemd template unit) の仕組みは構築済みだが、当面は VL-2B 一本で運用。

## systemd テンプレートユニット

モデル切り替えの仕組みは整備済み:

```bash
# テキスト専用 LLM に切り替え
sudo systemctl start axllm-serve@qwen3-1.7B-ax650

# VLM に戻す
sudo systemctl start axllm-serve@qwen3-vl-2B-Int4-ax650-ctx4095
```

- テンプレート: `/etc/systemd/system/axllm-serve@.service`
- sudoers NOPASSWD 設定済み
- 注意: `Conflicts=axllm-serve@*.service` のワイルドカードが効かない場合あり、手動で stop が必要

## 日時

2026-03-25 実施
