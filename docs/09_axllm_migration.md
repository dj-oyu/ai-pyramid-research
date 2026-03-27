# StackFlow → axllm 移行ガイド

## 移行概要

M5Stack StackFlowの推論サービス(llm_llm, llm_vlm, llm-openai-api)を
AXERA-TECH公式の`axllm`に移行。VLM推論速度が3.9→9.2 tokens/secに向上。

## 移行前後の比較

| 指標 | StackFlow | axllm |
|------|-----------|-------|
| VLMテキスト速度 | 3.9 tok/s | **9.1 tok/s** |
| VLM画像+テキスト速度 | 3.9 tok/s | **9.2 tok/s** |
| TTFT (テキスト) | 400ms | **316ms** |
| TTFT (画像) | 750ms | **352ms** |
| コンテキスト長 | 1,152 tokens | **3,584 tokens** |
| CMM使用量 | 3,582MB | 2,887MB |

## 現在の構成

```
axllm serve :8000        ← VLM/LLM推論 (メイン)
llm-sys                  ← CMM管理 (StackFlow残存)
ec_proxy                 ← EC制御 (独立)
```

### systemd サービス

| サービス | 状態 | 実行ユーザー | 備考 |
|---------|------|-------------|------|
| axllm-serve | **enabled, active** | root (非root化不可) | port 8000, VLMモデル |
| ax-proc-perms | **enabled** | root (oneshot) | NPU procfs権限設定 |
| llm-sys | enabled, active | root | CMM管理用に残存 |
| ec_proxy | enabled, active | root | ハードウェア制御 |
| llm-llm | **disabled** | - | 旧LLM推論 |
| llm-vlm | **disabled** | - | 旧VLM推論 |
| llm-openai-api | **disabled** | - | 旧OpenAI互換API |

> NPUデバイス権限の詳細は [12_npu_permissions.md](12_npu_permissions.md) を参照。

### モデル

| モデル | パス | サイズ | 用途 |
|--------|------|--------|------|
| Qwen3-VL-2B CTX4095 | `/opt/m5stack/data/qwen3-vl-2B-Int4-ax650-ctx4095/` | 3.1GB | VLM+テキスト (稼働中) |
| Qwen3-1.7B | `/opt/m5stack/data/qwen3-1.7B-ax650/` | 2.7GB | テキスト専用LLM (**削除済み** — ストレージ逼迫のため) |
| Qwen2.5-0.5B | `/opt/m5stack/data/qwen2.5-0.5B-Int4-ax650/` | 620MB | 旧テキストLLM |

## axllm インストール手順

```bash
# GitHub Actions CIからプリビルドバイナリを取得
gh run download -R AXERA-TECH/ax-llm <run_id> -n ax650-aarch64-artifacts -D /tmp/axllm-dl
sudo cp /tmp/axllm-dl/install/bin/axllm /usr/local/bin/axllm
sudo chmod +x /usr/local/bin/axllm

# 最新のrun_idを確認:
gh run list -R AXERA-TECH/ax-llm --limit 5
```

## systemd 登録

`/etc/systemd/system/axllm-serve.service`:
```ini
[Unit]
Description=axllm OpenAI-compatible API Server
After=network.target ax-proc-perms.service

[Service]
Type=simple
ExecStart=/usr/local/bin/axllm serve /opt/m5stack/data/qwen3-vl-2B-Int4-ax650-ctx4095/ --port 8000
Restart=always
RestartSec=3
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```

> axllmはrootで実行（カーネルドライバの制約で非root化不可）。
> NPUデバイス権限の詳細は [12_npu_permissions.md](12_npu_permissions.md) を参照。

```bash
sudo systemctl daemon-reload
sudo systemctl enable axllm-serve
sudo systemctl start axllm-serve
```

## StackFlowサービスの無効化

```bash
sudo systemctl disable llm-vlm llm-llm llm-openai-api
sudo systemctl stop llm-vlm llm-llm llm-openai-api
# llm-sys は残す (CMM管理)
```

## API互換性

axllm serveのOpenAI互換APIはStackFlowのAPIとほぼ互換。

**変更点:**
- モデルID: `qwen3-vl-2B-Int4-ax650` → `AXERA-TECH/Qwen3-VL-2B-Instruct-GPTQ-Int4-C256-P3584-CTX4095`
- pet-albumの `run_album` スクリプトで `VLM_MODEL` を更新済み
- `--vlm-model` 引数で起動時に指定可能

**互換のまま:**
- エンドポイント: `POST /v1/chat/completions`, `GET /v1/models`
- リクエスト形式: OpenAI Chat Completions API準拠
- 画像入力: `data:image/jpeg;base64,...` のインライン形式

## マルチモデル運用の制約

axllmは**1プロセス1モデル**。NPU (AX_ENGINE) は排他リソースのため、
2プロセス同時起動はCMMメモリ競合やNPUクラッシュのリスクがある。

**現在の運用:**
- Qwen3-VL-2Bでテキスト推論も兼用（9 tok/sで十分実用的）
- Qwen3-1.7B LLMは削除済み（ストレージ逼迫のため 2026-03-27 に削除）

**モデル切り替え方法:**
```bash
# VLM → LLM に切り替える場合
sudo systemctl stop axllm-serve
sudo sed -i 's|qwen3-vl-2B-Int4-ax650-ctx4095|qwen3-1.7B-ax650|' /etc/systemd/system/axllm-serve.service
sudo systemctl daemon-reload
sudo systemctl start axllm-serve

# 戻す場合は逆のsed
```

## Qwen3-1.7B 復旧手順

ストレージ逼迫のため 2026-03-27 に `/opt/m5stack/data/qwen3-1.7B-ax650/` を削除した。
復旧するには、AXERA-TECH の Pulsar2 `llm_build` で変換済みモデルを再取得する:

```bash
# 方法1: AXERA-TECH GitHub Actions CIからビルド済みアーティファクトを取得
# (変換済みモデルが公開されている場合)
gh run list -R AXERA-TECH/ax-llm --limit 10
gh run download -R AXERA-TECH/ax-llm <run_id> -n <artifact-name> -D /tmp/qwen3-1.7b-dl
sudo cp -r /tmp/qwen3-1.7b-dl /opt/m5stack/data/qwen3-1.7B-ax650

# 方法2: x86ホスト上で Pulsar2 Docker を使い自前変換
# (HuggingFace: Qwen/Qwen3-1.7B をダウンロード後)
pulsar2 llm_build \
  --input_path Qwen/Qwen3-1.7B/ \
  --output_path Qwen/qwen3-1.7B-ax650/ \
  --chip_type AX650 \
  --hidden_state_type bf16 \
  --kv_cache_type u8 \
  --parallel 8 \
  --max_token_len 4096
# 変換後のディレクトリをデバイスへ転送
scp -r Qwen/qwen3-1.7B-ax650/ admin-user@<device-ip>:/opt/m5stack/data/
```

> **注意:** VLM (Qwen3-VL-2B) がテキスト推論も兼用できるため、通常は復旧不要。
> テキスト専用LLMが必要な場面（速度重視等）でのみ復旧を検討。

## StackFlow完全復旧

axllmからStackFlowに戻す場合:
```bash
sudo systemctl stop axllm-serve
sudo systemctl disable axllm-serve
sudo systemctl enable llm-vlm llm-llm llm-openai-api
sudo systemctl start llm-sys llm-llm llm-vlm llm-openai-api
```
