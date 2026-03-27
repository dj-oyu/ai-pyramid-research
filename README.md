# AI Pyramid Pro チートシート

M5Stack AI Pyramid Pro (AX8850 SoC) のクイックリファレンス。
SSH ログイン後、すぐに VLM/TTS/ハードウェア操作を開始できます。

## デバイス概要

| 項目 | スペック |
|------|---------|
| SoC | Axera AX8850 (AX650C) |
| CPU | 8-core Cortex-A55 @ 1.5GHz |
| NPU | 24 TOPS @ INT8 |
| RAM | 8GB LPDDR4x (System ~2GB / NPU CMM 6GB) |
| OS | Ubuntu 22.04 aarch64 (headless) |
| VLM | Qwen3-VL-2B (9.2 tok/s) |
| TTS | kokoro.axera (RTF ~0.3, 約3倍速) |

## VLM (テキスト生成・画像認識)

axllm serve が port 8000 で常駐。OpenAI 互換 API。

```bash
# テキスト生成
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"AXERA-TECH/Qwen3-VL-2B-Instruct-GPTQ-Int4-C256-P3584-CTX4095",
       "messages":[{"role":"user","content":"猫について一文で教えて"}],
       "max_tokens":64}' | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"

# 画像認識 (base64)
IMG=$(base64 -w0 photo.jpg)
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"AXERA-TECH/Qwen3-VL-2B-Instruct-GPTQ-Int4-C256-P3584-CTX4095\",
       \"messages\":[{\"role\":\"user\",\"content\":[
         {\"type\":\"image_url\",\"image_url\":{\"url\":\"data:image/jpeg;base64,$IMG\"}},
         {\"type\":\"text\",\"text\":\"この画像を説明して\"}]}],
       \"max_tokens\":128}"

# モデル確認
curl -s http://localhost:8000/v1/models

# サービス管理
sudo systemctl status axllm-serve
sudo systemctl restart axllm-serve
```

## TTS (音声合成)

`kokoro-tts` コマンドで NPU 音声合成 → スピーカー再生。
実行中は axllm を自動停止し、完了後に自動再開します。

```bash
# 基本 (日本語、デフォルトボイス)
kokoro-tts --text "こんにちは、今日はいい天気ですね"

# 英語
kokoro-tts --text "Hello world" --lang a --voice /opt/m5stack/data/kokoro.axera/checkpoints/voices/af_heart.pt

# ボイス変更
kokoro-tts --text "テスト" --voice /opt/m5stack/data/kokoro.axera/checkpoints/voices/jm_kumo.pt

# ヘルプ
kokoro-tts --help
```

### 言語コード

| コード | 言語 | コード | 言語 |
|--------|------|--------|------|
| `j` | 日本語 (デフォルト) | `a` | American English |
| `z` | 中国語 | `b` | British English |
| `e` | スペイン語 | `f` | フランス語 |

### 日本語ボイス

| ファイル | 性別 | 備考 |
|---------|------|------|
| `jf_tebukuro.pt` | 女性 | デフォルト |
| `jf_alpha.pt` | 女性 | ニュートラル |
| `jf_gongitsune.pt` | 女性 | 「ごん狐」風 |
| `jf_nezumi.pt` | 女性 | 「ねずみの嫁入り」風 |
| `jm_kumo.pt` | 男性 | 「蜘蛛の糸」風 |

全ボイス一覧: `ls /opt/m5stack/data/kokoro.axera/checkpoints/voices/`

## VLM → TTS パイプライン

axllm でテキスト生成し、そのまま読み上げ。全てエッジデバイス上で完結。

```bash
TEXT=$(curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"AXERA-TECH/Qwen3-VL-2B-Instruct-GPTQ-Int4-C256-P3584-CTX4095",
       "messages":[{"role":"user","content":"猫が昼寝している様子を一文で描写して"}],
       "max_tokens":64,"temperature":0.7}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])")
echo "$TEXT"
kokoro-tts --text "$TEXT"
```

## ハードウェア制御 (ec_cli)

EC (組み込みコントローラ) 経由で LED、ファン、電源等を制御。**sudo 必須**。

```bash
# LED (48個 RGB)
sudo ec_cli-1.0 device --led_num -d '0'              # 全消灯
sudo ec_cli-1.0 device --led_num -d '48'             # 全点灯
sudo ec_cli-1.0 device --led_brightness -d '20'      # 明るさ (0-100)
sudo ec_cli-1.0 device --led_color -d '#ff0000'      # 色 (赤)

# ファン
sudo ec_cli-1.0 device --fan_speed                    # 現在の速度
sudo ec_cli-1.0 device --fan_speed -d '80'            # 速度設定 (0-100)

# 電源
sudo ec_cli-1.0 device --power_off                    # シャットダウン
sudo ec_cli-1.0 device --reboot                       # リブート

# システム情報
sudo ec_cli-1.0 device --version                      # ECファームウェア
sudo ec_cli-1.0 device --temp                         # CPU温度

# HDMI
sudo ec_cli-1.0 device --hdmi_status                  # HDMI状態
sudo ec_cli-1.0 device --hdmi_out_source -d '1'       # 出力ソース (0:HDMI-in, 1:SoC)

# ボタンイベント監視
sudo ec_cli-1.0 echo                                  # Ctrl+C で終了
```

詳細: [docs/06_ec_cli_reference.md](docs/06_ec_cli_reference.md)

## オーディオ

```bash
# スピーカーテスト
aplay /usr/share/sounds/alsa/Front_Center.wav

# 録音 (マイク)
arecord -D hw:0,1 -f S16_LE -r 16000 -c 1 -d 5 /tmp/rec.wav

# 再生
aplay /tmp/rec.wav

# 音量調整
amixer set 'DAC' 80%
```

## サービス一覧

| サービス | 状態 | 用途 |
|---------|------|------|
| `axllm-serve` | active | VLM/LLM推論 (port 8000) |
| `ax-proc-perms` | active (oneshot) | NPU権限設定 |
| `llm-sys` | active | CMM管理 |
| `ec_proxy` | active | ハードウェア制御 |

```bash
sudo systemctl status axllm-serve
sudo systemctl restart axllm-serve
```

## NPU の制約

- NPU は排他リソース: 2プロセス同時アクセスで SEGV
- `kokoro-tts` は axllm を自動停止→TTS→自動再開で管理
- axllm 再起動 (モデルロード) に約15秒かかる

## ファイル配置

```
/opt/m5stack/data/
├── qwen3-vl-2B-Int4-ax650-ctx4095/   # VLMモデル (3.1GB, 稼働中)
├── (qwen3-1.7B-ax650/)                # テキストLLM (2.7GB) ※ストレージ逼迫により削除済み
└── kokoro.axera/                       # TTS (554MB)
    ├── models/                         # NPUモデル
    ├── checkpoints/voices/             # 声紋ファイル
    └── demo_kokoro_ax.py               # 推論スクリプト

/usr/local/bin/
├── axllm                              # LLM推論エンジン
└── kokoro-tts                         # TTS ラッパースクリプト

/usr/local/m5stack/bin/
├── ec_cli-1.0                         # ハードウェア制御CLI
└── ec_proxy-1.0                       # ECプロキシデーモン
```

## 詳細ドキュメント

| ドキュメント | 内容 |
|------------|------|
| [01_hardware_specs.md](docs/01_hardware_specs.md) | ハードウェア諸元 |
| [02_inference_framework.md](docs/02_inference_framework.md) | StackFlow推論フレームワーク解析 |
| [03_pulsar2_guide.md](docs/03_pulsar2_guide.md) | Pulsar2モデル変換 |
| [04_qwen35_deployment.md](docs/04_qwen35_deployment.md) | Qwen 3.5 VLMデプロイ検討 |
| [05_optimal_inference.md](docs/05_optimal_inference.md) | 最適なAI推論 |
| [06_ec_cli_reference.md](docs/06_ec_cli_reference.md) | ec_cli全コマンドリファレンス |
| [07_device_tools_complete.md](docs/07_device_tools_complete.md) | 全デバイスツール・TCP API |
| [08_desktop_removal.md](docs/08_desktop_removal.md) | デスクトップ環境削除と復旧 |
| [09_axllm_migration.md](docs/09_axllm_migration.md) | StackFlow→axllm移行ガイド |
| [10_tts_comparison.md](docs/10_tts_comparison.md) | TTS選択肢の比較 |
| [11_audio_devices.md](docs/11_audio_devices.md) | オーディオデバイスとTTS連携 |
| [12_npu_permissions.md](docs/12_npu_permissions.md) | NPU権限・kokoro-tts セットアップ |
