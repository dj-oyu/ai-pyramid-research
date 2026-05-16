# AI Pyramid Pro TTS 選択肢の比較

## 候補一覧

| TTS | NPU活用 | 日本語 | サイズ | RTF | 備考 |
|-----|---------|--------|--------|-----|------|
| **kokoro.axera** | NPU | ja/en/zh | 554MB | **0.067** | AXERA公式、C++バイナリあり |
| **Piper-Plus** | CPU only | ja/en/zh/es/fr/pt | ~100MB | ~0.2 | OSSフォーク、ONNX |
| MeloTTS (StackFlow) | NPU | en/ja/zh | 60MB CMM | ~0.3推定 | プリインストール済み |
| CosyVoice (StackFlow) | CPU主体 | zh/en/ja/ko | 759MB | 不明 | プリインストール済み |
| Qwen3-TTS-1.7B | NPU | 10言語 | 2.6GB | 不明 | ランタイム未対応、VoiceDesign対応 |

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

### 5. Qwen3-TTS-12Hz-1.7B-VoiceDesign-AX650（未検証、ランタイム未対応）

AXERA-TECHがAX650向けにPulsar2で変換したQwen3-TTS。2026-03-19頃アップロード、最終更新 2026-03-27。

**上流モデル:** [Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign)（2026-01-22リリース、月間643Kダウンロード）

**特徴:**
- 10言語対応（中/英/日/韓/独/仏/露/葡/西/伊）
- VoiceDesign: 自然言語テキストで声質・感情・韻律を指示（参照音声不要）
- 元モデルのストリーミング遅延: 97ms
- 論文: [arXiv:2601.15621](https://arxiv.org/abs/2601.15621)

**AX650変換モデル構成 (~2.6GB):**
- `talker/` (~2.35GB): 28層分割 `.axmodel` (各~61MB) + text_embedding (622MB) + codec_embedding (12.6MB)
- `code-predictor/` (~250MB): 5層 `.axmodel` + 15個の lm_head + codec embeddings

**現状 (2026-03-30時点):**
- README は空。使い方のドキュメントなし
- **ax-llm**: Qwen3-TTS 対応は未実装（最新コミット 2026-03-27）
- **ax_tts_api**: Kokoro TTS のみ対応。Qwen3-TTS 統合は未着手
- コミュニティでの議論・動作報告なし
- モデルファイルは変換済みだが、**推論ランタイムが未対応のため実行不可**

**懸念:**
- 1.7Bパラメータ → CMM使用量が大きい（推定1.5GB+）
- axllmのVLM (CMM ~2.9GB) と同時にCMMに載るか不明（6GB上限）
- ランタイム対応待ちのステージング段階

**結論:** kokoro.axera の上位互換になり得る（多言語・VoiceDesign）が、ランタイム対応待ち。ax-llm または ax_tts_api に統合されるまで実機では使えない。

- HuggingFace: https://huggingface.co/AXERA-TECH/Qwen3-TTS-12Hz-1.7B-VoiceDesign-AX650
- 上流GitHub: https://github.com/QwenLM/Qwen3-TTS

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

---

## 追記: 実使用感とコミュニティ評価 (2026-05 リサーチ)

Reddit (r/LocalLLaMA), Hacker News, GitHub Issues, 各種レビュー記事から収集した実使用感の整理。

### kokoro 評価まとめ

#### 👍 ポジティブ評価
- **TTS Arena leaderboard #1** (2026年1月、82M パラメータで MOS 4.2)
  - XTTS-v2 (467M)・MetaVoice (1.2B) 等の大型モデルを撃破
- "af_heart" 等の英語ボイスは **非常にクリーン**
- **「音節が突然増える失敗」が起きない** (他TTSにありがちな破綻が少なく信頼性高い)
- 句読点・ポーズ処理が優秀、自然な区切り
- CPUでもほぼリアルタイム動作 (このデバイスは NPU で RTF 0.067)
- 日本語/韓国語が小サイズの割に良好

#### 👎 ネガティブ評価
- 「**抑揚が大半のフレーズで不自然**」「機械的なモジュレーション感が残る」(HN コメント)
- **感情表現が乏しい**、フラットな読み上げになりがち
- **ボイスクローン非対応** (XTTS-v2/ElevenLabs のような複製不可)
- **短文 (単語1個など) で品質が落ちる、長文向き** ← 会話アシスタント用途では地味に問題
- 中国語/日本語で **phonemizer起因の品質問題** を報告するユーザーあり
- 多言語ボイスは長文合成で時々不安定
- ローカルインストール後も HuggingFace への接続を試みるバグ報告あり (Kokoro-TTS-Local Issue #23)

#### 会話アシスタント用途での運用ヒント
- 短文応答 (「はい」「了解」等) の品質に注意 → LLM プロンプトで多少冗長な返答に整形すると良い
- 感情を出したい場面ではテキスト側の工夫が必要 (LLMで感嘆符・繰り返しを誘導)

### MeloTTS 補足

- **GitHub Issue #241**: 英語で "chokin" → "cokin"、"plugin" → "ploogin" など **発音不安定バグ** あり
- 日本語の specific な評価情報は少ない (議論薄 = ユーザー基盤が薄い)
- 軽量・低遅延は実証済み (AX650 RTF 0.125)
- kokoro と比べ **コミュニティ熱量・継続的議論が少ない** → トラブル時のサポート薄
- StackFlow 版 (プリインストール、CMM 60MB) と **AXERA-TECH/MeloTTS** (HF、独立利用可) は別物
- pip 依存スタックが kokoro より大幅に軽い (spacy/jieba/unidic_lite/misaki 等不要)
  - kokoro 削除して MeloTTS へ移行すれば pip 500MB+ 節約可能

### 6. Irodori-TTS (このデバイスでは非対応)

Flow Matching ベースの日本語特化 TTS。zero-shot voice cloning と絵文字感情制御が特徴。

**アーキテクチャ:**
- Rectified Flow Diffusion Transformer (RF-DiT) over continuous DACVAE latents
- 500M params (v3 では 2.5B あり)、F32 で約 2GB
- llm-jp/llm-jp-3-150m 初期化のテキストエンコーダ
- Semantic-DACVAE-Japanese-32dim で 48kHz 出力

**このデバイスでの実行性:**
- ❌ **CPU推論**: Ryzen 7 9700X で 5秒音声に 90秒 (RTF 18)。A55@1.5GHz では推定 15-45分/5秒で実用不可
- ❌ **NPU移植**: Flow Matching の反復サンプリング + 複雑な DiT を pulsar2 で axmodel 化する実績なし。公式 axmodel 配布なし
- ❌ **依存スタック**: torch≥2.10.0, torchcodec, numba 等。CUDA 12.8 前提

**コミュニティでの既知問題:**
- 公式が **「漢字読みが弱い、事前にひらがな変換推奨」** と明言 (G2P が脆弱)
- 音声品質自体は高評価

**結論:** 候補から除外。やりたければ PC で生成 + ストリーム再生のみ現実的。

- GitHub: https://github.com/Aratako/Irodori-TTS
- HuggingFace: https://huggingface.co/Aratako/Irodori-TTS-500M-v3
- ライセンス: MIT

### Qwen3-TTS 続報 (2026-05 時点)

`AXERA-TECH/Qwen3-TTS-12Hz-1.7B-VoiceDesign-AX650` と `0.6B-Base-AX650` 両方を再調査:

- **README が空** (31 バイト、初期コミットのみ、使用方法ドキュメントなし)
- 1.7B-VoiceDesign: Apache-2.0、`code-predictor/` + `talker/` (~2.6GB)
- 0.6B-Base: MIT、おそらく参照音声によるボイスクローン版
- **ランタイム未対応** (ax-llm/ax_tts_api への統合待ち、変更なし)
- 上流 Qwen3-TTS は日本語含む10言語、VoiceDesign で自然言語による声質指定可
- 3-6ヶ月後に再評価する候補

### 日本語TTSの構造的限界 (全モデル共通課題)

> **日本語TTSの自然さの上限は G2P (grapheme-to-phoneme) レイヤーで決まる**、モデルアーキテクチャではない (lilting.ch)

- MeCab系の形態素解析: 文脈無視・辞書1択読み
- 全モデル共通の弱点: **固有名詞・当て字 (ateji)・新語/スラング**
- 大規模多言語モデル (CosyVoice2等) は subword tokenizer + 大規模事前学習で end-to-end 学習し、G2P 依存を回避する方向
- 小型モデル (kokoro等) でこれを補うには: **テキスト前処理で読み仮名を明示** するのが現実解

### 追加候補: Style-Bert-VITS2 / AivisSpeech (未検証)

VITS系の軽量派生、日本語アシスタント向けに評判が良い:

- **CPU動作** (NPU不要)、Mac/Windows/Linux 対応
- 数値による感情・スタイル制御 (Bert-VITS2 系統)
- OpenJTalk + MeCab 使用 (日本語特化)
- 「**ローカルアシスタント・常時稼働に実用的**」評価多数 (lilting.ch)
- AX650 axmodel は未確認、CPU負荷検証が必要
- 候補としてリスト入りさせる価値あり (将来 NPU 移植実績が出れば本命候補に)

### まとめ表 (拡張版)

| 優先度 | TTS | 理由 |
|--------|-----|------|
| 1 | **kokoro.axera** | NPU高速(RTF 0.067)、日本語対応、C++バイナリ、TTS Arena #1 (2026-01)、ただし短文と感情表現に弱 |
| 2 | **Piper-Plus** | NPU競合なし、日本語品質高、軽量、ストリーミング対応 |
| 2.5 | **AXERA-TECH/MeloTTS** (HF版) | NPU動作 (RTF 0.125)、pip依存軽い、ただし発音バグ報告あり |
| 3 | Qwen3-TTS | 将来有望だが未成熟、ランタイム待ち |
| - | Style-Bert-VITS2/AivisSpeech | CPU動作、日本語アシスタント向け実績、AX650未検証 |
| - | Irodori-TTS | 高品質だがこのデバイスで実行不可、漢字読み弱 |
| - | MeloTTS (StackFlow) | StackFlow依存、単独利用不可 |
| - | CosyVoice | 重い、GPU向け、削除候補 (CosyVoice3 AX650版は英語専用で日本語非対応) |

### 参考リンク (2026-05 リサーチ)

- [Kokoro TTS Review (ReviewNexa)](https://reviewnexa.com/kokoro-tts-review/)
- [Hacker News: Kokoro discussion](https://news.ycombinator.com/item?id=45114884)
- [VoxCPM2 and OSS TTS 2026 (lilting.ch)](https://lilting.ch/en/articles/voxcpm2-tokenizer-free-local-tts)
- [MeloTTS pronunciation Issue #241](https://github.com/myshell-ai/MeloTTS/issues/241)
- [Best TTS Models 2026 (CodeSOTA)](https://www.codesota.com/guides/tts-models)
- [ElevenLabs Alternatives 2026 (ocdevel)](https://ocdevel.com/blog/20250720-tts)
- [Kokoro-TTS-Local Issue #23 (network connection issue)](https://github.com/PierrunoYT/Kokoro-TTS-Local/issues/23)
