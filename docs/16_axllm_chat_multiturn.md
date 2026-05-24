# ax-llm チャット機能・マルチターン会話ガイド

最終更新: 2026-05-24

## 概要

ax-llm `1d3d90f`（2026-05-14、PR #34 でマージ）でQwen3.5向けのマルチターン会話が実装された。
現在デバイスに入っているバイナリ（2026-05-24更新）にはこの機能が含まれている。

---

## マルチターン会話の仕様

### 基本的な使い方

OpenAI 互換 API の `messages` 配列に複数ターンを積んで渡す。

```python
history = []

# ターン1
history.append({"role": "user", "content": "私の猫の名前はまぐろです。"})
reply = chat(history)
history.append({"role": "assistant", "content": reply})  # ← そのまま保存

# ターン2
history.append({"role": "user", "content": "まぐろの好きな場所はどこでしょう？"})
reply = chat(history)
history.append({"role": "assistant", "content": reply})
```

### 重要: assistant メッセージは加工禁止

ax-llm はサーバー側で KV キャッシュを保持し、**クライアントが返す history との整合性をチェックする**。
`<think>...</think>` ブロックを削って history に入れると以下のエラーが返る：

```
检测到历史被修改，无法复用已有图像 KV。
请保持返回的 history 继续追加，或 /reset 后重新开始。
```

（「履歴が改変されています。KV キャッシュを再利用できません」）

**ルール: `reply` は一切加工せずそのまま history に追加する。**  
UI 表示用に `<think>` ブロックを削除したい場合は、表示専用の変数を別途用意する。

```python
reply = chat(history)
display = re.sub(r"<think>.*?</think>", "", reply, flags=re.DOTALL).strip()  # 表示用
history.append({"role": "assistant", "content": reply})  # 保存は元のまま
```

---

## 実測値（Qwen3.5-2B-CTX8K、2026-05-24）

| 項目 | 値 |
|------|-----|
| モデル | Qwen3.5-2B-AX650-GPTQ-Int4-C256-P6K-CTX8K |
| コンテキストウィンドウ | 8,192 tokens |
| 応答速度 | 13.4 tok/s |
| 4ターン累計トークン（概算） | ~445 / 8192（5%程度） |
| 記憶再現テスト（名前・年齢・性別）| 4ターン後に正確に再現 ✓ |

`<think>` ブロックはトークンを消費するが、数十ターン程度では CTX8K を圧迫しない。

---

## アラビア文字混入バグと対処

### 現象

`Qwen3.5-2B` GPTQ Int4 量子化版で、日本語・英語出力に `ربة` などアラビア文字が混入する。

```
例: "猫がソファでربةくつろいでいます。"
```

### 原因

GPTQ Int4 量子化による特定トークンの重み劣化。NPU の確率計算レベルの問題であり、
プロンプトで禁止しても完全には防げない。Axera 公式の修正待ち。

### 対処（smart-pet-camera PR #214、2026-05-24）

`vlm/mod.rs` に `strip_arabic()` 関数を追加し、全 VLM 呼び出し後に適用：

```rust
fn strip_arabic(text: &str) -> String {
    text.chars()
        .filter(|&c| !('\u{0600}'..='\u{06FF}').contains(&c))
        .collect()
}
```

適用箇所: `analyze` / `analyze_with_detections` / `summarize_day` の3箇所。

### フィルタ範囲の検証結果

- U+0600–U+06FF（Arabic ブロック）で既知サンプル `ربة`（REH/BEH/TEH MARBUTA）を完全カバー
- 日本語・英語への誤爆なし
- 挿入型の混入パターンでは除去後も日本語として成立する
- 唯一の注意点: 助詞間への挿入（例: `猫がربةを`）を除去すると `猫がを` になる（稀）

### システムプロンプトでの緩和

プロンプト禁止は頻度を下げる補助として有効だが、重み劣化レベルの問題には根治にならない。
フィルタと組み合わせて使う。

```
# VLM_SYSTEM_PROMPT に追加
"Use only English letters, numbers, and standard punctuation — never Arabic, Chinese, or other non-Latin characters."

# DAY_SUMMARY_SYSTEM に追加
"2. 日本語のみで書く。中国語・英語・ローマ字・アラビア文字・その他の非日本語文字を混ぜない。"
```

---

## VLM hot-swap（daily summary）

### 現在の設定（2026-05-24）

Gemma モデルを削除したため、`/opt/smart-pet-camera/.env` の hot-swap 設定を無効化：

```bash
# PET_ALBUM_VLM_SWAP_VISION_UNIT=...
# PET_ALBUM_VLM_SWAP_TEXT_UNIT=...
# PET_ALBUM_VLM_SWAP_TEXT_MODEL=...
```

daily summary は起動中の 2B モデルをそのまま使用する（中断なし）。

### hot-swap を復活させる条件

テキスト専用の高品質モデル（Gemma 相当）を導入した場合に再有効化を検討。
同一モデルへの hot-swap はサービス再起動（~15秒）が発生するだけで無意味。

---

## 既知の挙動・注意事項

| 項目 | 内容 |
|------|------|
| thinking モード | 2B では `/no_think` で品質が下がる傾向あり。think ブロックを出させた方が良い |
| `usage` フィールド | API レスポンスの `usage.prompt_tokens` 等は常に 0（未実装）|
| CMM stickiness | `systemctl stop` しても NPU CMM は解放されない。モデル切り替えにはリブート必要 |
| `<think>` ブロックの漏れ | `strip_think()` で除去。ax-llm 側の既知挙動 |
| 日次サマリーの温度 | `temperature=0.3`（VLM キャプションは `0.1`）|
