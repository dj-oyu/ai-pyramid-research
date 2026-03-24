# AI Pyramid Pro 調査レポート

M5Stack AI Pyramid Pro (AX8850 SoC) のハードウェア・ソフトウェア調査と、高性能VLM推論の実現方法をまとめたリポジトリ。

## ドキュメント構成

| ドキュメント | 内容 |
|------------|------|
| [01_hardware_specs.md](docs/01_hardware_specs.md) | ハードウェア諸元（コマンド調査結果） |
| [02_inference_framework.md](docs/02_inference_framework.md) | StackFlow推論フレームワーク完全解析 |
| [03_pulsar2_guide.md](docs/03_pulsar2_guide.md) | Pulsar2モデル変換フレームワークの使い方 |
| [04_qwen35_deployment.md](docs/04_qwen35_deployment.md) | Qwen 3.5 VLMのデプロイ検討 |
| [05_optimal_inference.md](docs/05_optimal_inference.md) | 最適なAI推論アプリケーション・高性能VLM推論 |
| [06_ec_cli_reference.md](docs/06_ec_cli_reference.md) | ec_cliハードウェア制御コマンドリファレンス（実測値付き） |
| [07_device_tools_complete.md](docs/07_device_tools_complete.md) | 全デバイスツール・StackFlow TCP API・procfs・OTA |

## デバイス要約

- **SoC**: Axera AX8850 (chip: AX650C, board: AX650N_M5stack_8G)
- **CPU**: 8-core Cortex-A55 @ 1.5GHz
- **NPU**: 24 TOPS @ INT8, Axera Neutron アーキテクチャ
- **RAM**: 8GB LPDDR4x (System ~2GB / NPU CMM 6GB)
- **OS**: Ubuntu 22.04.5 LTS, Linux 5.15.73 aarch64
- **USB PD**: 15V / 1.25A (18.75W)

## 主要な発見

1. **推論フレームワーク**: M5Stack「StackFlow」フレームワーク（C++ネイティブ + ZMQ + FastAPI）
2. **StackFlow TCP API (port 10001)**: KWS→ASR→LLM→TTSのパイプラインを直接構築可能
3. **OpenAI互換API (port 8000)**: StackFlowのラッパー。VLM/LLM/TTS/ASR対応
4. **ec_cli**: LED(48個)/ファン/電源/LCD/USB/PCIe/HDMI等のハードウェア制御（要sudo）
5. **ax-llm**: AXERA-TECH公式の代替推論フレームワーク（`axllm serve`で高速推論）
6. **AXERA-TECH HuggingFace**: 150以上のaxmodel変換済みモデル公開中
7. **VLM性能**: qwen3-vl-2B で TTFT ~400ms, ~3.9 token/s, ビジョン入力384×384固定
8. **ストレージ制約**: eMMC残り2GB、M.2 SSD増設が必須（PCIeスロット2基空き）
