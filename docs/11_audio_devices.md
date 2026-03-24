# AI Pyramid Pro オーディオデバイスとTTS連携

## オーディオハードウェア

### サウンドカード一覧

| カード | デバイス | コーデック | 方向 | 用途 |
|--------|---------|-----------|------|------|
| card 0, dev 0 | `hw:0,0` | ES8311 HiFi (I2S) | 再生 | **本体スピーカー** |
| card 0, dev 1 | `hw:0,1` | ES7210 4CH ADC (I2S) | 録音 | **本体マイク** (4ch) |
| card 1, dev 0 | `hw:1,0` | I2S HiFi | 再生 | HDMI Audio 1 |
| card 2, dev 0 | `hw:2,0` | I2S HiFi | 再生 | HDMI Audio 2 |
| card 3, dev 0 | `hw:3,0` | USB Audio | 録音 | USB3 Video (HDMI IN) |

### コーデック詳細

- **スピーカー**: Everest Semiconductor ES8311 — 低消費電力モノラルDACコーデック
- **マイク**: Everest Semiconductor ES7210 — 4チャンネルADC（マイクアレイ対応）
- **接続**: I2S (2033000.i2s_mst)

### ALSA デバイスパス

```
/dev/snd/pcmC0D0p   ← 本体スピーカー (PLAYBACK)
/dev/snd/pcmC0D1c   ← 本体マイク (CAPTURE)
/dev/snd/pcmC1D0p   ← HDMI Audio 1 (PLAYBACK)
/dev/snd/pcmC2D0p   ← HDMI Audio 2 (PLAYBACK)
/dev/snd/pcmC3D0c   ← USB HDMI IN Audio (CAPTURE)
/dev/snd/controlC0  ← ミキサー制御
```

## セットアップ

### 必須: audioグループへの追加

`/dev/snd/` デバイスは `root:audio` グループ所有。sudo無しで使うには:

```bash
sudo usermod -aG audio admin-user
# 再ログインで反映
```

### 必須: ALSA デフォルトカード設定

デスクトップ環境(pulseaudio)を削除した場合、ALSA のデフォルトカード設定が必要:

```bash
sudo tee /etc/asound.conf << 'EOF'
defaults.pcm.card 0
defaults.pcm.device 0
defaults.ctl.card 0
EOF
```

この設定がないと `aplay` 等で `Unknown PCM default` エラーになる。

## 基本操作

### 再生

```bash
# 本体スピーカーで再生
aplay audio.wav

# デバイス明示指定
aplay -D hw:0,0 audio.wav

# HDMI出力
aplay -D hw:1,0 audio.wav

# rawデータのパイプ再生
cat audio.raw | aplay -r 24000 -f S16_LE -c 1
```

### 録音

```bash
# 本体マイクで録音 (16kHz, 16bit, モノラル, 5秒)
arecord -D hw:0,1 -f S16_LE -r 16000 -c 1 -d 5 recording.wav
```

### テスト音

```bash
# ビープ音生成 (440Hz, 1秒)
python3 -c "
import wave, struct, math
sr = 16000
with wave.open('/tmp/beep.wav', 'w') as w:
    w.setnchannels(1); w.setsampwidth(2); w.setframerate(sr)
    w.writeframes(b''.join(
        struct.pack('<h', int(32767 * 0.5 * math.sin(2 * 3.14159 * 440 * i / sr)))
        for i in range(sr)
    ))
"
aplay /tmp/beep.wav
```

## TTS → スピーカー連携

### kokoro.axera (推奨)

#### CLIで直接再生

```bash
# WAVファイル生成 → aplayで再生
cd /opt/m5stack/data/kokoro.axera
sudo ./cpp/install_ax650/kokoro \
  -d models -l j -v checkpoints/voices/jm_kumo.pt \
  -t "こんにちは、今日はいい天気ですね" \
  -o /tmp/tts_out.wav

aplay /tmp/tts_out.wav
```

#### HTTPサーバー経由

```bash
# TTSサーバー起動 (port 28000)
sudo ./cpp/install_ax650/kokoro_srv -p 28000 -l j \
  -d models --voice_path checkpoints/voices -v jm_kumo

# 別ターミナルから音声生成+再生
curl -s -X POST "http://localhost:28000/tts" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "sentence=猫がごはんを食べています" \
  -o /tmp/tts.wav && aplay /tmp/tts.wav
```

#### Python API連携

```python
import requests
import subprocess

def speak(text, lang="j", voice="jm_kumo"):
    resp = requests.post("http://localhost:28000/tts", data={
        "sentence": text,
        "language": lang,
    })
    if resp.status_code == 200:
        with open("/tmp/tts.wav", "wb") as f:
            f.write(resp.content)
        subprocess.run(["aplay", "/tmp/tts.wav"])

# VLM結果を読み上げ
speak("猫がソファの上で寝ています")
```

### Piper-Plus (CPU-only代替)

```bash
# インストール
pip install piper-tts-plus

# 推論+再生 (パイプで直接再生)
echo "こんにちは" | piper-plus \
  --model ja_JP-tsukuyomi-medium.onnx \
  --output-raw | aplay -r 22050 -f S16_LE -c 1

# WAVファイル出力
echo "猫を検知しました" | piper-plus \
  --model ja_JP-tsukuyomi-medium.onnx \
  --output /tmp/alert.wav
aplay /tmp/alert.wav
```

### aplayの代替: sox (play コマンド)

```bash
# soxのインストール
sudo apt install -y sox

# 再生（サンプルレート変換が自動）
play /tmp/tts.wav

# 音量調整付き再生
play /tmp/tts.wav vol 0.5
```

## VLM + TTS パイプライン例

ペットカメラで猫を検知 → VLMで分析 → TTSで読み上げ:

```python
import requests, subprocess, base64, json

# 1. VLMで画像分析
with open("photo.jpg", "rb") as f:
    b64 = base64.b64encode(f.read()).decode()

vlm_resp = requests.post("http://localhost:8000/v1/chat/completions", json={
    "model": "AXERA-TECH/Qwen3-VL-2B-Instruct-GPTQ-Int4-C256-P3584-CTX4095",
    "messages": [{"role": "user", "content": [
        {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64}"}},
        {"type": "text", "text": "この写真の猫が何をしているか、日本語で一文で説明して"}
    ]}],
    "max_tokens": 50
})
caption = vlm_resp.json()["choices"][0]["message"]["content"]
print(f"VLM: {caption}")

# 2. TTSで読み上げ
tts_resp = requests.post("http://localhost:28000/tts", data={"sentence": caption})
with open("/tmp/caption.wav", "wb") as f:
    f.write(tts_resp.content)
subprocess.run(["aplay", "/tmp/caption.wav"])
```

## 注意事項

- スピーカーは**モノラル**（ES8311はシングルチャンネルDAC）
- マイクは**4チャンネル**対応（ES7210）だが、通常はモノラル録音で使用
- `sudo` が必要な場合は `audioグループ` に追加すれば不要になる
- HDMI Audio (card 1/2) はHDMIケーブル接続時のみ使用可能
- kokoro_srvとaxllm serveは同時に動作可能（別ポート、NPU共有）
