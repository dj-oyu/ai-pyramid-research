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
| [08_desktop_removal.md](docs/08_desktop_removal.md) | デスクトップ環境削除と復旧・Wayland移行 |
| [09_axllm_migration.md](docs/09_axllm_migration.md) | StackFlow→axllm移行ガイド（性能2.4倍向上） |

## 現在のデバイス構成

- **SoC**: Axera AX8850 (chip: AX650C, board: AX650N_M5stack_8G)
- **CPU**: 8-core Cortex-A55 @ 1.5GHz
- **NPU**: 24 TOPS @ INT8, Axera Neutron アーキテクチャ
- **RAM**: 8GB LPDDR4x (System ~2GB / NPU CMM 6GB)
- **OS**: Ubuntu 22.04.5 LTS, Linux 5.15.73 aarch64 (headless)
- **推論**: axllm serve (Qwen3-VL-2B CTX4095, 9.2 tok/s)
- **ストレージ空き**: 5.9GB / 29GB

## 主要な成果

1. **推論速度2.4倍向上**: StackFlow (3.9 tok/s) → axllm (9.2 tok/s)
2. **コンテキスト長3倍**: 1,152 → 3,584 tokens
3. **ストレージ7GB解放**: snap/Docker/デスクトップ削除 + Rustキャッシュ整理
4. **ec_cli完全リファレンス**: 48個RGB LED、ファン、電源、PCIe等の全コマンド実測値付き
5. **StackFlow完全解析**: ZMQ通信、ネイティブバイナリ構造、TCP API仕様を解明
6. **AXERA-TECHエコシステム調査**: Pulsar2、ax-llm、150+モデルの評価
