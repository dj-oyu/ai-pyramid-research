# AI Pyramid Pro TTS 選択肢の比較

## 候補一覧

| TTS | NPU活用 | 日本語 | サイズ | RTF | 備考 |
|-----|---------|--------|--------|-----|------|
| **kokoro.axera** | NPU | ja/en/zh | 554MB | **0.067** | AXERA公式、C++バイナリあり |
| **Piper-Plus** | CPU only | ja/en/zh/es/fr/pt | ~100MB | ~0.2 | OSSフォーク、ONNX |
| MeloTTS (StackFlow) | NPU | en/ja/zh | 60MB CMM | ~0.3推定 | プリインストール済み |
| CosyVoice (StackFlow) | CPU主体 | zh/en/ja/ko | 759MB | 不明 | プリインストール済み |
| Qwen3-TTS-1.7B | NPU | 10言語 | 2.4GB | 不明 | 最新、ドキュメント未整備 |

## 詳細評価

### 1. kokoro.axera（最推奨）

AXERA-TECH公式のNPU最適化TTS。Kokoro TTSのAX650移植版。

**性能 (AX650N実測):**
- RTF: **0.067**（4.8秒の音声を0.32秒で生成 = 15倍速）
- 初回初期化: ~5秒
- 推論内訳: Model1(22ms) + Model2(17ms) + Model3(185ms) + Model4-ONNX(74ms) = 322ms

**メモリ:**
- 中国語: OS 233MB + CMM 237MB = 470MB
- 英語: OS 23MB + CMM 237MB = 260MB

**対応言語:** 中国語(zh), 英語(en), 日本語(ja)

**音声:** 複数話者（zf_xiaoyi, af_heart, jm_kumo等）

**実行方法:**
- Python: `python demo_kokoro_ax.py --text "..." --lang j --voice jm_kumo.pt`
- C++バイナリ: `./kokoro` (CLI) / `./kokoro_srv` (HTTPサーバー :8080)
- HTTP API: `POST /tts` でWAV返却

**利点:**
- NPU活用で圧倒的に高速（RTF 0.067）
- C++バイナリ提供（Python不要で動作可能）
- HTTPサーバーモード内蔵
- axllm serveと共存可能（別ポート）

**懸念:**
- axllm serveとNPU/CMMを共有。同時推論時の競合リスク
- CMM 237MB追加使用（VLMの2,887MBと合わせて~3.1GB、6GB上限内）
- ダウンロード 554MB

**インストール:**
```bash
# ダウンロード
sudo python3 -c "
from huggingface_hub import snapshot_download
snapshot_download('AXERA-TECH/kokoro.axera', local_dir='/opt/m5stack/data/kokoro.axera', local_dir_use_symlinks=False)
"

# C++バイナリで実行
sudo /opt/m5stack/data/kokoro.axera/cpp/install_ax650/kokoro_srv -p 28000 -l j
```

- GitHub参考: https://github.com/ml-inory/melotts.axera
- HuggingFace: https://huggingface.co/AXERA-TECH/kokoro.axera

---

### 2. Piper-Plus（CPU-only代替）

Piper TTSの日本語強化フォーク。VITS + OpenJTalk。NPU不要。

**性能:**
- RTF: ~0.2（Cortex-A55 CPUのみ、NPU不使用）
- 短いテキストは1秒以内で処理

**対応言語:** 日本語, 英語, 中国語, スペイン語, フランス語, ポルトガル語（6言語, 571話者）

**日本語機能:**
- OpenJTalk統合（アクセント解析）
- プロソディマーカー（A1/A2/A3）
- カスタム辞書（200+技術用語）

**モデル形式:** ONNX

**実行方法:**
```bash
# インストール
pip install piper-tts-plus

# 推論
echo "こんにちは" | piper-plus --model ja_JP-tsukuyomi-medium.onnx --output hello.wav

# ストリーミングモード
piper-plus --model ... --output-raw | aplay -r 22050 -f S16_LE
```

**利点:**
- NPUに一切触らない → axllm serveと完全に競合しない
- 日本語品質が高い（OpenJTalk + WavLM discriminator訓練）
- WebAssembly版もあり（ブラウザで動作）
- 軽量（モデル~50-100MB）
- ストリーミング対応

**懸念:**
- CPUのみなので8コアCortex-A55で速度が限定的（RTF ~0.2）
- NPU活用のkokoro(RTF 0.067)より3倍遅い
- 推論中のCPU負荷がVLM推論のCPU側処理に影響する可能性

- GitHub: https://github.com/ayutaz/piper-plus

---

### 3. MeloTTS（StackFlow プリインストール）

M5Stack StackFlowに組み込み済みのTTS。libtts.so (19MB) として提供。

**性能 (AX650):**
- エンコーダ: 30-123ms
- デコーダスライス: 各92-108ms
- RTF: ~0.3推定

**メモリ:** CMM 60MB（非常に軽量）

**対応言語:** 英語, 日本語, 中国語

**実行方法:**
- StackFlow経由のみ（llm_llm + llm_sysが必要）
- OpenAI API (`/v1/audio/speech`) またはTCP :10001プロトコル

**利点:**
- 既にインストール済み、追加ダウンロード不要
- CMM使用量が極小（60MB）
- StackFlowパイプラインに統合可能（KWS→ASR→LLM→TTS）

**懸念:**
- StackFlowのllm_llmサービスが必要（axllm移行後は停止中）
- axllm serveと共存させるにはStackFlow LLMを再起動する必要があり、NPU競合リスク
- 単独で使用不可（StackFlow依存）

---

### 4. CosyVoice（StackFlow プリインストール）

Alibaba FunAudioLLMのCosyVoice。`/opt/m5stack/share/cosy-voice/` に759MBのPythonパッケージ。

**性能:**
- ストリーミングモードで150ms遅延（GPU環境）
- ARM CPU環境での性能データなし、おそらく遅い

**対応言語:** 中国語, 英語, 日本語, 韓国語

**懸念:**
- 759MBのストレージ使用（主にPython依存パッケージ）
- GPU向け設計、NPU非対応
- CPU-onlyだと実用的な速度が出ない可能性
- StackFlow依存

**結論:** このデバイスでは非推奨。削除候補。

---

### 5. Qwen3-TTS-12Hz-1.7B-VoiceDesign-AX650（最新、未検証）

AXERA-TECHがAX650向けに変換したQwen3-TTS。19時間前に更新（2026-03-24時点）。

**特徴:**
- 10言語対応（中/英/日/韓/独/仏/露/葡/西/伊）
- 3秒音声クローニング
- 感情表現制御
- 元モデルのストリーミング遅延: 97ms

**サイズ:** 2.4GB（ダウンロード）

**懸念:**
- READMEが空、ドキュメント未整備（出たばかり）
- 1.7Bパラメータ → CMM使用量が大きい（推定1.5GB+）
- axllmのVLMと同時にCMMに載るか不明
- 動作実績なし

**結論:** 有望だが時期尚早。ドキュメントが整備されてから検討。

- HuggingFace: https://huggingface.co/AXERA-TECH/Qwen3-TTS-12Hz-1.7B-VoiceDesign-AX650

---

## 推奨の組み合わせ

### パターンA: NPU活用（最高速）

```
axllm serve :8000     ← VLM/LLM推論 (CMM ~2.9GB)
kokoro_srv :28000     ← TTS (CMM ~237MB)
                        合計CMM: ~3.1GB / 6GB上限
```

同時推論時のNPU競合リスクがあるが、TTSとVLMが同時に推論するケースは少ない（TTS中はVLMリクエストが来ない）。

### パターンB: NPU/CPU分離（安全）

```
axllm serve :8000     ← VLM/LLM推論 (NPU)
piper-plus            ← TTS (CPU only、NPU不使用)
```

NPU競合リスクゼロ。速度はkokoroより遅いがCPU処理なので安全。

### パターンC: 将来（Qwen3-TTS待ち）

```
axllm serve :8000     ← VLM/LLM + TTS? (もし統合されれば)
```

axllmがTTSモデルもserveできるようになれば最もシンプル。

## まとめ

| 優先度 | TTS | 理由 |
|--------|-----|------|
| 1 | **kokoro.axera** | NPU高速(RTF 0.067)、日本語対応、C++バイナリ、HTTPサーバー内蔵 |
| 2 | **Piper-Plus** | NPU競合なし、日本語品質高、軽量、ストリーミング対応 |
| 3 | Qwen3-TTS | 将来有望だが未成熟 |
| - | MeloTTS | StackFlow依存、単独利用不可 |
| - | CosyVoice | 重い、GPU向け、削除候補 |
