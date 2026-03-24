# AI Pyramid Pro - ハードウェア諸元

## デバイス概要

M5Stack AI Pyramid Pro は、Axera AX8850 SoC を搭載したエッジAIコンピューティングボックス。

## SoC: Axera AX8850 (内部チップID: AX650C)

| 項目 | 値 |
|------|-----|
| チップ型番 | AX8850 (chip_type: AX650C_CHIP) |
| CPU | ARM Cortex-A55 x8 コア |
| CPU周波数 | 1200 - 1500 MHz |
| NPU | Axera Neutron アーキテクチャ |
| NPU性能 | 24 TOPS @ INT8 |
| ISP | 8K@30fps 対応 |
| VPU | H.264/H.265 エンコード/デコード、16ch 1080p 並列デコード |

### AX650N との違い

- AX650N: 43.2 TOPS@INT4 / 10.8 TOPS@INT8（上位チップ）
- AX8850: 24 TOPS@INT8（本デバイス搭載チップ）
- ソフトウェアスタック・SDK は共通（pulsar2, ax-llm 等）

## メモリ

| 項目 | 値 |
|------|-----|
| 物理RAM | 8GB LPDDR4x |
| システム用 | ~2GB (`MemTotal: 1949908 kB`) |
| NPU/HW用 CMM | 6144MB (6GB) |
| Swap | 4GB |

### CMM メモリ詳細（コマンド調査結果）

```
cat /proc/ax_proc/mem_cmm_info
SDK VERSION: ax_cmm V3.6.4_20250822020158
PARTITION: Size=6291456KB(6144MB), NAME="anonymous"
現在使用量: Cur=3126MB
最大使用量: Max=4904MB
空き容量: Free=9757MB（累積解放含む）
```

## ストレージ

| 項目 | 値 |
|------|-----|
| eMMC | 29.1GB (mmcblk0) |
| ルートパーティション | 29GB（使用率94%、残り2.0GB） |
| M.2スロット | M-KEY 2242/2230 x2（SSD拡張用） |

## インターフェース

- Ethernet: デュアルギガビット
- USB: USB 3.0 + USB Type-C
- HDMI: デュアル HDMI 2.0 (入力1 + 出力1)
- M.2: M-KEY 2242/2230 x2

## 電源

- PD アダプタ DC 9V@3A (27W) 以上必須
- DC 5V では起動不可

## OS

| 項目 | 値 |
|------|-----|
| OS | Ubuntu 22.04.5 LTS (Jammy) |
| カーネル | Linux 5.15.73 aarch64 SMP PREEMPT |
| アーキテクチャ | aarch64 |

## NPU デバイスノード

```
/dev/npu              # NPUデバイス
/dev/ax_base          # ベースデバイス
/dev/ax_cmm           # CMMメモリ管理
/dev/ax_proton        # NPUランタイム
/dev/ax_vdec          # ビデオデコーダ
/dev/ax_jdec          # JPEGデコーダ
/dev/ax_vdsp0/1       # DSPプロセッサ
/dev/ax_pyra_lite0    # Pyramid Lite
/dev/ax_pool          # メモリプール
```

## ハードウェア制御: ec_proxy

```
/usr/local/m5stack/bin/ec_proxy-1.0   # EC Proxyデーモン
/usr/local/m5stack/bin/ec_cli-1.0     # ECコマンドラインツール
```

ec_proxyはM5Stackの組み込みコントローラ(EC)と通信し、LED、ファン、電源管理などを制御する。
詳細: https://docs.m5stack.com/en/stackflow/ai_pyramid/ec_proxy

## 調査に使用したコマンド

```bash
# CPU情報
cat /proc/cpuinfo
lscpu

# メモリ
free -h
cat /proc/meminfo

# ストレージ
df -h
lsblk

# OS情報
uname -a
cat /etc/os-release

# NPU/チップ情報
cat /proc/ax_proc/chip_type
cat /proc/ax_proc/mem_cmm_info
cat /proc/ax_proc/version
ls /dev/ax_* /dev/npu

# デバイスツリー
dmesg | grep -i "ax650\|npu"
```
