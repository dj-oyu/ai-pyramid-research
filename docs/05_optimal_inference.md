# 最適なAI推論アプリケーション・高性能VLM推論

## 現在の推論性能

### 既存フレームワーク（M5Stack ModuleLLM）

- 推論エンジン: `llm_llm-1.12` / `llm_vlm-1.11`（M5Stack独自バイナリ）
- APIサーバー: FastAPI (Python) → TCP socket → ネイティブエンジン
- 現用モデル: `qwen3-vl-2B-Int4-ax650`（メモリ3,582MB使用）

推論速度の参考値（AX650N/AX8850プラットフォーム全般）:
- SmolLM2-360M: ~26 tokens/sec
- Qwen3-0.6B (w8a16): ~13 tokens/sec
- 一般的なLLMユーザー報告: ~20 tokens/sec

## 推論速度向上の選択肢

### 1. ax-llm フレームワーク（最も推奨）

GitHub: https://github.com/AXERA-TECH/ax-llm

M5Stackの`llm_llm`/`llm_vlm`とは別の、AXERA-TECH公式のLLM推論フレームワーク。

**利点:**
- OpenAI互換APIサーバー内蔵 (`axllm serve`)
- 最新モデルへの対応が早い
- CLIインタラクティブモード
- 直接NPU呼び出し（Python仲介レイヤーなし）

**対応モデル:**
- LLM: Qwen2.5, Qwen3, MiniCPM, SmolLM2, Llama3, HY-MT1.5-1.8B
- VLM: Qwen3-VL-2B, SmolVLM2-500M-Video, FastVLM-1.5B, InternVL3_5-1B

**導入方法:**
```bash
git clone https://github.com/AXERA-TECH/ax-llm.git
cd ax-llm
./install.sh
axllm serve /path/to/model --port 8000
```

### 2. モデルサイズの最適化

より小さいモデルに切り替えることで速度向上:

| モデル | パラメータ数 | 期待速度 | VLM対応 |
|--------|-----------|---------|---------|
| SmolVLM2-500M-Video | 500M | 高速 | 動画対応 |
| FastVLM-0.5B | 500M | 最高速 | 画像対応 |
| FastVLM-1.5B | 1.5B | 高速 | 画像対応 |
| InternVL3_5-1B | 1B | 高速 | 画像対応 |
| Qwen3-VL-2B (現在) | 2B | ベースライン | 画像対応 |
| Qwen3-VL-4B | 4B | やや遅い | 画像対応、高精度 |

### 3. 量子化レベルの調整

- **W4A16 (INT4)**: 最もメモリ効率が良い、速度も最速
- **W8A16 (INT8)**: やや高品質、メモリ・速度は劣る
- 現在のモデルは INT4 で最適化済み

### 4. KVキャッシュ長の最適化

短いKVキャッシュ = 速度向上 + メモリ節約（ただしコンテキスト長が制限される）

現在のqwen3-vl設定: `max_context_length: 1152`

## 高性能VLMの推奨候補

### このデバイスに最適なVLM（性能/速度バランス）

1. **FastVLM-1.5B-GPTQ-Int4** - Apple研究チームのモデル、速度重視の設計
   - HuggingFace: https://huggingface.co/AXERA-TECH/FastVLM-1.5B-GPTQ-Int4
   - メモリ使用量が少なく高速

2. **InternVL3_5-1B** - OpenGVLab の高性能コンパクトVLM
   - HuggingFace: https://huggingface.co/AXERA-TECH/InternVL2_5-1B

3. **SmolVLM2-500M-Video-Instruct** - HuggingFace製、動画理解対応
   - HuggingFace: https://huggingface.co/AXERA-TECH/SmolVLM2-500M-Video-Instruct
   - 500Mパラメータで超高速

4. **Qwen3-VL-4B-Instruct-GPTQ-Int4** - 精度重視の場合
   - HuggingFace: https://huggingface.co/AXERA-TECH/Qwen3-VL-4B-Instruct-GPTQ-Int4
   - 現在の2Bモデルからの精度向上

5. **MiniCPM4-V** - 面白法人の軽量VLM
   - AXERA-TECHのHugging Faceに公開済み

### 新たに登場した注目モデル

- **Qwen3-VL-2B-Instruct-GPTQ-Int4-C256-P3584-CTX4095**: 長コンテキスト版（3日前更新）
  - コンテキスト4095トークン対応、現在の1152から大幅に拡張

- **Qwen3-TTS-12Hz-1.7B-VoiceDesign-AX650**: 音声合成（19時間前更新）

## 推奨アクション

### 即座にできること

1. **ax-llm のインストール** → 既存のM5Stackフレームワークと並行テスト
2. **FastVLM-1.5B のダウンロード** → 速度重視のVLM評価
3. **Qwen3-VL-2B の長コンテキスト版** → コンテキスト長の改善

### ストレージの準備

現在のeMMCは残り2GBしかないため、M.2 SSD の増設を強く推奨。
モデルファイルは2-4GB/モデルのため、複数モデルを試すにはSSDが必須。

## 関連プロジェクト・リソース

- ax-llm: https://github.com/AXERA-TECH/ax-llm
- ax-pipeline (CV): https://github.com/AXERA-TECH/ax-pipeline
- ax-npu-kit-650: https://github.com/AXERA-TECH/ax-npu-kit-650
- libclip.axera: https://github.com/AXERA-TECH/libclip.axera
- AXERA-TECH HuggingFace: https://huggingface.co/AXERA-TECH
