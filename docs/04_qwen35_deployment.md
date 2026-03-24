# Qwen 3.5 VLM のデプロイ検討

## Qwen 3.5 モデルファミリー（2026年2-3月リリース）

### フラッグシップモデル
- **Qwen3.5-397B-A17B**: MoE、Gated Delta Networksアーキテクチャ
- **Qwen3.5-122B-A10B**: MoE
- **Qwen3.5-35B-A3B**: MoE（INT4量子化で〜3GB）
- **Qwen3.5-27B**: Dense

### Smallモデルシリーズ（2026年3月リリース）
- **Qwen3.5-9B**: INT4で〜6GB VRAM
- **Qwen3.5-4B**: INT4で〜3GB VRAM、Visual Agent対応
- **Qwen3.5-2B**: INT4で〜2GB未満
- **Qwen3.5-0.8B**: INT4で〜1GB未満、スマートフォン動作可

### 主な特徴
- テキスト+ビジョンを統合した早期融合学習（Early Fusion）
- 201言語・方言対応
- Qwen3-VLを超えるマルチモーダル性能
- エージェント指向設計

## AI Pyramid Pro での実行可能性分析

### デバイスの制約

| リソース | 利用可能 |
|---------|---------|
| NPU CMMメモリ | 6,144MB (6GB) |
| NPU性能 | 24 TOPS@INT8 |
| システムRAM | ~2GB |
| ストレージ空き | ~2GB（要SSD拡張） |

### 実行可能なモデルサイズ

| モデル | INT4サイズ | 実行可否 | 備考 |
|--------|----------|---------|------|
| Qwen3.5-0.8B | <1GB | 可能 | 余裕あり |
| Qwen3.5-2B | <2GB | 可能 | 推奨 |
| Qwen3.5-4B | ~3GB | 要検証 | CMMに収まるが余裕少 |
| Qwen3.5-9B | ~6GB | 困難 | CMM上限ギリギリ、他のHW用途と競合 |
| Qwen3.5-27B以上 | >15GB | 不可能 | メモリ不足 |

### 現実的なターゲット: Qwen3.5-2B または Qwen3.5-4B

## デプロイ方法

### 方法1: AXERA-TECH公式の変換済みモデルを使用（推奨）

AXERA-TECHのHugging Faceには以下のVLMモデルが公開済み:

| モデル | サイズ | URL |
|--------|-------|-----|
| Qwen3-VL-2B-Instruct-GPTQ-Int4 | 2B | https://huggingface.co/AXERA-TECH/Qwen3-VL-2B-Instruct-GPTQ-Int4 |
| Qwen3-VL-4B-Instruct-GPTQ-Int4 | 4B | https://huggingface.co/AXERA-TECH/Qwen3-VL-4B-Instruct-GPTQ-Int4 |
| Qwen3-VL-8B-Instruct-GPTQ-Int4 | 8B | https://huggingface.co/AXERA-TECH/Qwen3-VL-8B-Instruct-GPTQ-Int4 |
| InternVL2_5-1B | 1B | https://huggingface.co/AXERA-TECH/InternVL2_5-1B |
| FastVLM-1.5B-GPTQ-Int4 | 1.5B | https://huggingface.co/AXERA-TECH/FastVLM-1.5B-GPTQ-Int4 |
| SmolVLM2-500M-Video-Instruct | 500M | https://huggingface.co/AXERA-TECH/SmolVLM2-500M-Video-Instruct |
| MiniCPM4-V | - | https://huggingface.co/AXERA-TECH |

> **注意:** Qwen**3.5**-VLの変換済みモデルは2026-03-24時点で未公開。Qwen**3**-VLモデルが最新。
> Qwen3.5は統合アーキテクチャのため、Qwen3-VLとは変換方法が異なる可能性あり。

### 方法2: Pulsar2で自前変換

Qwen3.5系はPulsar2のllm_buildの検証済みリストに含まれていないため、リスクあり。
Qwen3系（Qwen3-0.6B, Qwen3-1.7B, Qwen3-4B）はAXERA-TECHが変換実績あり。

```bash
# x86ホスト上でPulsar2 Docker環境にて実行
pulsar2 llm_build \
  --input_path Qwen/Qwen3.5-2B/ \
  --output_path Qwen/Qwen3.5-2B-w4a16/ \
  --kv_cache_len 1023 \
  --hidden_state_type bf16 \
  --weight_type s4 \
  --prefill_len 128 \
  --chip AX650
```

### 方法3: ax-llm フレームワークを使用

```bash
# デバイス上で実行
git clone https://github.com/AXERA-TECH/ax-llm.git
cd ax-llm
./install.sh  # または ./build_ax650.sh

# CLIモード
axllm run <model_path>

# OpenAI互換APIサーバー
axllm serve <model_path>  # port 8000
```

## 推奨アクションプラン

1. **即座に試せる**: AXERA-TECHの `Qwen3-VL-2B-Instruct-GPTQ-Int4` をダウンロードし、既存の `qwen3-vl-2B-Int4-ax650` と差し替えて性能比較
2. **高性能VLMを追求**: `Qwen3-VL-4B-Instruct-GPTQ-Int4` の導入を検討（CMMメモリに収まるか要検証）
3. **Qwen3.5対応待ち**: AXERA-TECHがQwen3.5-VLの変換済みモデルを公開するのを待つ
4. **ax-llm導入**: 既存のM5Stack推論フレームワークよりも高速な推論が期待できる
