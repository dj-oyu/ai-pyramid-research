# Pulsar2 モデル変換ガイド

## Pulsar2 とは

Axera独自開発のニューラルネットワークコンパイラ。変換・量子化・コンパイル・異種演算を統合した4-in-1ツール。
Hugging Face形式のモデル（`.safetensor` / `pytorch_model.bin`）を `.axmodel` 形式に変換してNPU上で実行可能にする。

- 公式ドキュメント: https://pulsar2-docs.readthedocs.io/en/latest/
- GitHub: https://github.com/AXERA-TECH/pulsar2-docs-en
- ローカルドキュメント: https://github.com/AXERA-TECH/pulsar2-docs-en

## 対象チップ

| チップ | Pulsar2での指定 |
|-------|----------------|
| AX650N / AX8850 | `--chip AX650` |
| AX630C | `--chip AX630C` |
| AX620E | `--chip AX620E` |

## LLMモデル変換: `pulsar2 llm_build`

### 検証済みモデルアーキテクチャ

Llama2/3/3.2, TinyLlama-1.1B, Qwen1.5/2/2.5, Phi2/3, MiniCPM, MiniCPM-V 2.0, SmolLM, ChatGLM3, OpenBuddy

> **注意:** Pulsar2 V3.2時点で「実験的機能」の位置づけ。Qwen3/3.5系は公式検証リストに未記載だが、AXERA-TECHのHugging Face上にQwen3系の変換済みモデルが公開されている。

### 変換手順

#### 1. 環境準備

```bash
# Pulsar2 Docker環境（x86ホストマシン上で実行）
# ax-llm-buildリポジトリのclone
git clone https://github.com/AXERA-TECH/ax-llm-build.git
cd ax-llm-build
```

#### 2. モデルダウンロード

```bash
pip install -U huggingface_hub
huggingface-cli download --resume-download Qwen/Qwen2-0.5B-Instruct --local-dir Qwen/Qwen2-0.5B-Instruct
```

#### 3. モデルコンパイル

```bash
pulsar2 llm_build \
  --input_path Qwen/Qwen2-0.5B-Instruct/ \
  --output_path Qwen/Qwen2-0.5B-w8a16/ \
  --kv_cache_len 1023 \
  --hidden_state_type bf16 \
  --prefill_len 128 \
  --chip AX650
```

#### 4. 埋め込みファイルの抽出・最適化

```bash
chmod +x ./tools/fp32_to_bf16
chmod +x ./tools/embed_process.sh
./tools/embed_process.sh Qwen/Qwen2-0.5B-Instruct/ Qwen/Qwen2-0.5B-w8a16/
```

### 主要パラメータ

| パラメータ | 説明 | 推奨値 |
|-----------|------|--------|
| `--chip` | ターゲットチップ | `AX650`（AI Pyramid Pro） |
| `--hidden_state_type` | 隠れ状態の精度 | `bf16` |
| `--weight_type` / `-w` | 重みの量子化 | `s8`(INT8) or `s4`(INT4) |
| `--kv_cache_len` | KVキャッシュ長 | 1023 |
| `--prefill_len` | プリフィル長 | 128 |
| `--parallel` | 並列ビルド数 | 8（ホストCPUコア数に応じて） |
| `--post_weight_type` | ポストレイヤー量子化 | `bf16` or `s8` |

### 出力ファイル構成

```
output_path/
├── model.embed_tokens.weight.bfloat16.bin  # 埋め込み重み（必須）
├── qwen2_p128_l0_together.axmodel          # レイヤー0（必須）
│   ... (各レイヤー)
├── qwen2_p128_lN_together.axmodel          # 最終レイヤー（必須）
└── qwen2_post.axmodel                      # ポストプロセス（必須）
```

### コンパイル所要時間の目安

- Qwen2-0.5B: 約6分（Xeon Gold 6336Y, 32GB RAM）
- モデルサイズに比例して増加

## CNNモデル変換: `pulsar2 build`

ONNX形式のCNN/Visionモデルを `.axmodel` に変換するコマンド。YOLOv5, MobileNet等。
LLM以外のビジョンモデル変換に使用。

## Pulsar2の入手

AXERA-TECHのHugging Faceからダウンロード可能:
https://huggingface.co/AXERA-TECH (Pulsar2リポジトリ)

Docker イメージとして提供される。
