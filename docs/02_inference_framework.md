# AI Pyramid Pro - 推論フレームワーク解析

## フレームワーク名: StackFlow

M5Stack独自の組み込みAI推論フレームワーク。
GitHub: https://github.com/m5stack/StackFlow

### 設計思想
- 分散通信アーキテクチャ: 各ユニット(LLM, VLM, TTS, ASR等)が独立プロセスとして動作
- ユニット間通信: **ZeroMQ (ZMQ)** メッセージング
- 外部インターフェース: TCP (port 10001) + JSON RPC
- 実装言語: **C++** (GCC 10.3 ARM A-profile)
- ビルドパス: `/home/m5stack/Workspace/StackFlow/SDK/`

## アーキテクチャ全体図

```
[ユーザー / アプリケーション]
       │
       ▼
[OpenAI互換API :8000]          ← llm_openai_api-1.10 (launcher) → FastAPI (Python)
       │ TCP socket :10001
       ▼
┌──────────────────────────────────────────────────────┐
│  StackFlow デーモン群（ネイティブ C++ バイナリ）         │
│                                                      │
│  llm_sys-1.6   ← システム管理・CMM監視・モデル管理     │
│      │ ZMQ                                           │
│      ├── llm_llm-1.12  ← LLMテキスト推論エンジン      │
│      └── llm_vlm-1.11  ← VLM画像+テキスト推論エンジン  │
│                                                      │
│  [tokenizer_*.py]  ← トークナイザーHTTPサーバー         │
└──────────────────────────────────────────────────────┘
       │ AX_ENGINE API
       ▼
[Axera NPU Hardware]  ← .axmodel を直接実行
       │
       ▼
[/soc/lib/ SDK ライブラリ群]
```

## プロセス一覧（稼働中）

| PID | プロセス | 起動日 | 役割 |
|-----|---------|-------|------|
| 511 | `/opt/m5stack/bin/llm_sys-1.6` | Feb17 | システム管理デーモン |
| 543 | `/opt/m5stack/bin/llm_llm-1.12` | Feb17 | LLMテキスト推論 |
| 570 | `/opt/m5stack/bin/llm_vlm-1.11` | Feb17 | VLM画像+テキスト推論 |
| 563 | `python3 api_server.py` | Feb17 | OpenAI互換APIサーバー |
| 2201 | `tokenizer_qwen2.5-0.5B-*.py :8080` | Feb17 | Qwen2.5トークナイザー |
| 3587304 | `tokenizer_qwen3-vl-2B-*.py :8090` | Mar21 | Qwen3-VLトークナイザー |

## コンポーネント詳細

### 1. llm_openai_api-1.10（ランチャー）

- パス: `/opt/m5stack/bin/llm_openai_api-1.10`
- サイズ: 15KB（極小）
- 役割: **Python api_server.py を起動するだけのラッパー**
- 内部で `execvp("python3", "/opt/m5stack/bin/ModuleLLM-OpenAI-Plugin/api_server.py")` を実行
- `PYTHONPATH=/opt/m5stack/lib/openai-api/site-packages` を設定

### 2. api_server.py（OpenAI互換APIサーバー）

- 実装: `/opt/m5stack/bin/ModuleLLM-OpenAI-Plugin/api_server.py`
- フレームワーク: **FastAPI** (Python)
- サービス: `llm-openai-api.service`

**対応エンドポイント:**
- `GET /v1/models` - モデル一覧
- `POST /v1/chat/completions` - チャット補完（ストリーミング対応）
- `POST /v1/audio/transcriptions` - 音声認識
- `POST /v1/audio/speech` - 音声合成

**バックエンドディスパッチ:**

| モデルタイプ | バックエンド | 通信方式 |
|------------|-----------|---------|
| `llm` | `LlmClientBackend` | TCP :10001 → llm_llm |
| `vlm` | `VisionModelBackend` | TCP :10001 → llm_vlm |
| `tts` | `TtsClientBackend` | TCP :10001 → llm_llm |
| `asr` | `ASRClientBackend` | TCP :10001 → llm_llm |

**メモリ管理:**
- `ModelDispatcher` がCMMメモリの残量を監視
- メモリ不足時は使用中モデルを自動アンロードして新モデルのためにメモリを確保
- `MemoryChecker` → `SYSClient.cmminfo()` で `/proc/ax_proc/mem_cmm_info` 相当の情報取得

### 3. llm_sys-1.6（システム管理デーモン）

- サイズ: 2.2MB
- サービス: `llm-sys.service`（最初に起動、他の全サービスの前提）
- 依存: `libzmq.so.5`, `libstdc++.so.6`
- 役割:
  - CMMメモリ情報の提供 (`cmminfo` アクション)
  - モデル一覧の管理 (`lsmode` アクション)
  - ハードウェア情報の提供 (`hwinfo` アクション)
  - ZMQハブとしてユニット間通信を仲介

**提供API (TCP :10001 JSON RPC):**
- `cmminfo` - NPU CMMメモリ使用状況
- `axcl_cmminfo` - AXCL CMMメモリ情報
- `hwinfo` - ハードウェア情報
- `lsmode` - インストール済みモデル一覧

### 4. llm_llm-1.12（LLMテキスト推論エンジン）

- サイズ: 4.4MB
- サービス: `llm-llm.service` (llm-sys.service の後に起動)
- ソースパス: `main_llm/src/runner/`

**依存ライブラリ:**
```
libax_engine.so      ← NPU推論エンジン本体
libax_interpreter.so ← モデルインタプリタ
libax_sys.so         ← システムAPI (メモリ割当等)
libzmq.so.5          ← ZeroMQ通信
```

**NPU API呼び出しフロー:**
```
AX_SYS_Init()
  → AX_ENGINE_Init()
  → AX_ENGINE_CreateHandle(axmodel)
  → AX_ENGINE_CreateContext() / CreateContextV2()
  → AX_ENGINE_GetGroupIOInfo()
  → AX_ENGINE_RunGroupIOSync()  ← 推論実行
  → AX_ENGINE_DestroyHandle()
AX_SYS_Deinit()
```

**対応モデルアーキテクチャ（内蔵トークナイザー）:**

| トークナイザークラス | 対応モデル |
|-------------------|----------|
| `TokenizerLLaMa` | Llama2/3, TinyLlama |
| `TokenizerQwen` | Qwen1.5/2/2.5/3 |
| `TokenizerMINICPM` | MiniCPM |
| `TokenizerPhi3` | Phi-3 |

> 注: 実際のトークナイザーは外部Pythonプロセス(`tokenizer_type: 2`)も使用可能。
> 現在の設定では外部HTTP (`localhost:8080`) を使用。

**推論機能:**
- KVキャッシュプリフィル (`GenerateKVCachePrefill`)
- 動的axmodelレイヤーロード (`b_dynamic_load_axmodel_layer`) - メモリ節約のため
- 埋め込みのmmapロード (`b_use_mmap_load_embed`)
- サンプリングパラメータ: `temperature`, `top_p`, `top_k`, `repetition_penalty`
- EOS検出で自動停止、token/s計測ログ出力

### 5. llm_vlm-1.11（VLM画像+テキスト推論エンジン）

- サイズ: 8.3MB
- サービス: `llm-vlm.service` (llm-sys.service の後に起動)
- ビルド元: `/home/m5stack/Workspace/StackFlow/SDK/`

**追加機能（llm_llmとの差分）:**
- ビジョンエンコーダ(`image_encoder_axmodel`)のロード・実行
- VPM(Visual Perception Module) / Resampler対応 (`filename_vpm_resampler_axmodedl`)
- 画像入力のNHWCフォーマット処理
- 動画入力対応（`b_video`, `temporal_patch_size`, `fps`）
- JPEG/PNG/WebM等の画像デコード（OpenCV内蔵、OpenJPEG内蔵）

**対応VLMアーキテクチャ:**
- `qwen3` (Qwen3-VL)
- `qwen2.5_vl` (Qwen2.5-VL) - NHWCのみサポート
- `minicpmv` (MiniCPM-V) - VPM Resampler使用
- `internvl3` (InternVL3)

### 6. トークナイザープロセス

モデル設定の `ext_scripts` で指定された外部Pythonスクリプト:

| スクリプト | ポート | モデル |
|-----------|-------|--------|
| `tokenizer_qwen2.5-0.5B-Int4-ax650.py` | 8080 | Qwen2.5-0.5B |
| `tokenizer_qwen3-vl-2B-Int4-ax650.py` | 8090 | Qwen3-VL-2B |

HuggingFace transformersのAutoTokenizerをHTTPサーバーとして公開。
llm_llm/llm_vlm はトークンIDの埋め込み→NPU推論を担当し、テキスト↔トークンID変換はトークナイザーに委譲。

## 通信プロトコル詳細

### 外部 → StackFlow (TCP :10001 JSON RPC)

**リクエスト:**
```json
{
  "request_id": "uuid",
  "work_id": "llm|vlm|sys",
  "action": "setup|inference|exit|pause|cmminfo|lsmode|hwinfo",
  "object": "llm.setup|llm.utf-8|vlm.setup|vlm.jpeg.base64|...",
  "data": { ... }
}
```

**レスポンス:**
```json
{
  "request_id": "uuid",
  "work_id": "...",
  "error": {"code": 0, "message": ""},
  "data": { ... }
}
```

**ストリーミング推論レスポンス:**
```json
{
  "request_id": "uuid",
  "data": {
    "delta": "生成されたトークンテキスト",
    "finish": false
  }
}
```

### 内部 StackFlow ユニット間 (ZMQ)

- `llm_sys` がZMQハブとして機能
- 設定: `config_zmq_min_port`, `config_zmq_max_port` で動的ポート割当
- シリアル(UART)経由のZMQ通信もサポート (`config_serial_zmq_port`)

## モデル設定ファイル

### `/opt/m5stack/data/models/mode_<モデル名>.json`

モデルごとのNPU推論パラメータを定義。これがStackFlowコアエンジンの設定。

**Qwen3-VL-2B の設定例 (抜粋):**

```json
{
  "mode": "qwen3-vl-2B-Int4-ax650",
  "type": "vlm",
  "mode_param": {
    "tokenizer_type": 2,
    "url_tokenizer_model": "http://localhost:8080",
    "filename_tokens_embed": "model.embed_tokens.weight.bfloat16.bin",
    "filename_post_axmodel": "qwen3_vl_text_post.axmodel",
    "template_filename_axmodel": "qwen3_vl_text_p128_l%d_together.axmodel",
    "filename_image_encoder_axmodel": "Qwen3-VL-2B-Instruct_vision.axmodel",
    "axmodel_num": 28,
    "tokens_embed_num": 151936,
    "tokens_embed_size": 2048,
    "b_use_mmap_load_embed": true,
    "enable_temperature": true,
    "temperature": 0.7,
    "enable_top_k_sampling": true,
    "top_k": 40,
    "vision_config.height": 384,
    "vision_config.width": 384,
    "vision_config.patch_size": 16,
    "vision_config.spatial_merge_size": 2,
    "cmm_size": 3582336
  }
}
```

### `/opt/m5stack/bin/ModuleLLM-OpenAI-Plugin/config/config.yaml`

OpenAI互換APIレイヤーの設定。`model_list.py` が `lsmode` APIで自動生成・更新。

## 利用可能モデル

| モデルID | タイプ | CMMメモリ | axmodel数 | embed_size | ディスク |
|---------|--------|----------|-----------|------------|--------|
| qwen2.5-0.5B-Int4-ax650 | LLM | 560MB | 24 layers | 896 | 620MB |
| qwen3-vl-2B-Int4-ax650 | VLM | 3,582MB | 28 layers + vision | 2048 | 2.6GB |
| melotts-en/ja/zh-ax650 | TTS | 60MB | - | - | - |
| sense-voice-small-10s-ax650 | ASR | - | - | - | - |
| sherpa-onnx-streaming-* | ASR | - | - | - | - |

### axmodelファイル詳細（Qwen3-VL-2B）

- テキストレイヤー: 28個 × 各45MB = 1,260MB
- ポストプロセス: 324MB
- ビジョンエンコーダ: 1個（サイズは別途）
- 埋め込み重み: bfloat16形式
- **合計ディスク: 約2GB（axmodelのみ）**

## NPU SDK ライブラリ (`/soc/lib/`)

```
libax_engine.so       ← NPU推論エンジン（AX_ENGINE_* API）
libax_interpreter.so  ← モデルインタプリタ
libax_sys.so          ← システムAPI（メモリ管理等）
libax_proton.so       ← NPUランタイム
libax_vdec.so         ← ビデオデコーダ
libax_venc.so         ← ビデオエンコーダ
libax_ivps.so         ← 画像ビデオ処理
libax_vo.so           ← ビデオ出力
libax_ive.so          ← 画像ビデオエンジン
libax_dsp.so          ← DSPプロセッサ
... (他多数)
```

## M5Stack ライブラリ (`/opt/m5stack/lib/`)

```
libMNN.so             ← MNN推論エンジン（Alibaba）
libonnxruntime.so.1   ← ONNX Runtime
libncnn.so            ← ncnn推論エンジン（Tencent）
libsherpa-ncnn-core.so ← sherpa音声認識
libtts.so             ← TTS
libzmq.so.5           ← ZeroMQ
libfst.so.16          ← OpenFst（音声認識用）
libglog.so.0          ← Google Logging
```

> 注: MNN, ONNX Runtime, ncnn は主にASR/TTSモデル用。LLM/VLMの推論はAX_ENGINE APIを直接使用。

## パッケージ管理

dpkgパッケージとしてモデルを管理:
```
llm-model-qwen2.5-0.5b-int4-ax650  0.4  arm64
llm-model-qwen3-vl-2b-int4-ax650   0.5  arm64
```

## StackFlowの制約と注意点

1. **閉鎖的なバイナリ**: llm_llm, llm_vlm, llm_sys はソースコード非公開のプリコンパイルバイナリ
2. **VLMの対応**: llm_vlm-1.11は `qwen3`, `qwen2.5_vl`, `minicpmv`, `internvl3` の4アーキテクチャのみ対応
3. **コンテキスト長制限**: VLMは`max_context_length: 1152`に制限（設定変更可能だがCMM制約あり）
4. **プールサイズ**: LLMは同時2リクエスト、VLMも2（メモリ許容範囲内）
5. **動的レイヤーロード**: `b_dynamic_load_axmodel_layer` で全レイヤーをCMMに常駐せず必要時ロード可能（速度とトレードオフ）
