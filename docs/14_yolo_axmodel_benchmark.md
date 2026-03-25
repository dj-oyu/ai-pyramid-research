# YOLO axmodel 検出ベンチマーク on AI Pyramid Pro (AX8850)

## 実験日: 2026-03-25

## 目的

AI Pyramid Pro (AX8850, NPU 24 TOPS) 上で YOLO26/YOLO11/YOLOv8 の axmodel を動かし、
ペットカメラ画像での cat 検出精度を比較。オフライン backfill での Level2 検出に最適なモデルを選定。

## 環境

- デバイス: M5Stack AI Pyramid Pro (AX8850 / AX650C)
- NPU: 24 TOPS INT8, AX_ENGINE v2.12.0s
- CMM: 6GB (VLM Qwen3-VL-2B が ~3.5GB 使用中)
- ランタイム: ax-samples (AXERA-TECH 公式サンプル)
- 閾値: `PROB_THRESHOLD = 0.10`, `NMS_THRESHOLD = 0.45`
- 入力: 640×640 letterbox JPEG (pet-album の crop_panel API 経由)

## テストモデル

| モデル | HuggingFace | サイズ | 推論時間 (NPU3) | CMM |
|--------|-------------|--------|----------------|-----|
| YOLO26n | AXERA-TECH/yolo26 | 2.6MB | 1.38ms | 3.3MB |
| YOLO26s | AXERA-TECH/yolo26 | 10MB | 3.17ms | 10.2MB |
| YOLO26m | AXERA-TECH/yolo26 | 22MB | 8.64ms | 27.6MB |
| YOLO26l | AXERA-TECH/yolo26 | 28MB | 11.17ms | 33.9MB |
| YOLO11s | AXERA-TECH/YOLO11 | — | 3.19ms | — |
| YOLO11x | AXERA-TECH/YOLO11 | — | — | — |
| YOLOv8s | AXERA-TECH/YOLOv8 | — | 3.59ms | — |

## テスト画像

### Image A: 明るい室内 (mike, 三毛猫)
- ファイル: `comic_20260325_191747_chatora.jpg` panel 0
- 猫: 三毛猫が餌皿の前でかがんでいる。明るい室内。
- RDK X5 (YOLO26n BPU): cat 検出あり

### Image B: 暗い室内 (chatora, キジトラ)
- ファイル: `comic_20260320_223257.jpg` panel 0
- 猫: キジトラが暗い部屋で丸まっている。低照度。
- RDK X5 (YOLO26n BPU): cat 22% で検出

## 結果

### Image A (明るい画像)

| モデル | 検出 | class | confidence | bbox |
|--------|------|-------|-----------|------|
| YOLO26n | pizza 35% | 誤検出 | — | — |
| YOLO26s | bowl 43% | bowl のみ | — | cat なし |
| YOLO26m | dog 64% | **誤分類** | cat→dog | [229,328,344,406] |
| **YOLO26l** | **cat 71%** | **正解** | 高 | [231,325,343,406] |
| **YOLO11s** | **cat 81%** | **正解・最高** | 最高 | [231,329,344,406] |
| YOLO11x | dog 50% | 誤分類 | cat→dog | [228,327,344,407] |
| YOLOv8s | dog 74% | 誤分類 | cat→dog | [227,326,345,406] |

### Image B (暗い画像)

| モデル | 検出 | class | confidence | bbox |
|--------|------|-------|-----------|------|
| YOLO26n | — | 検出ゼロ | — | — |
| YOLO26s | — | 検出ゼロ | — | — |
| YOLO26m | **cat 37%** | **正解** | 中 | [10,178,529,497] |
| **YOLO26l** | **cat 19%** | **正解** | 低 | [4,145,558,497] |
| YOLO11s | — | **検出ゼロ** | — | — |
| YOLO11x | dog 14% | 誤分類 | — | [102,236,531,491] |
| YOLOv8s | — | 検出ゼロ | — | — |

### 全画像パネルスキャン (comic_20260325_195004_chatora, 4パネル)

YOLO26m で 4パネル全て cat ゼロ。bowl(39%), book(15%), dining table は安定検出。
YOLO26n/s/m/l の全サイズで cat ゼロ（この画像の猫は検出困難な姿勢）。

## 分析

### cat/dog 誤分類問題

- COCO の cat(15) と dog(16) は隣接クラス
- axmodel の w8a16 量子化で classification head の精度が劣化
- YOLO26m, YOLOv8s, YOLO11x は猫を dog として検出する傾向
- **YOLO11s と YOLO26l は正しく cat と分類**

### モデル別特性

| モデル | 明るい画像 | 暗い画像 | 誤分類 | ノイズ | 総合評価 |
|--------|----------|---------|-------|-------|---------|
| YOLO26l | cat 71% ◎ | cat 19% △ | なし | 多め (10 det) | **バランス最良** |
| YOLO11s | cat 81% ◎ | 検出ゼロ ✗ | なし | 少ない (4 det) | **明所最強** |
| YOLO26m | dog 64% ✗ | cat 37% ○ | cat→dog | 中程度 | 暗所のみ |
| YOLOv8s | dog 74% ✗ | 検出ゼロ | cat→dog | 多め | 不推奨 |
| YOLO11x | dog 50% ✗ | dog 14% ✗ | cat→dog | 多め | 不推奨 |

### 推奨: デュアルモデル戦略

**YOLO11s + YOLO26l を両方走らせてマージ:**

- YOLO11s: 3.2ms, 明所で cat 81% (最高精度)
- YOLO26l: 11.2ms, 暗所でも cat 19% (唯一の検出)
- 合計 ~15ms — オフラインなら全く問題ない
- どちらかが cat/dog を検出すれば採用
- cat/dog は「pet」として統合扱い（家に犬はいない）

### 代替案: cat/dog 統合

家庭内ユースケースでは cat と dog の区別は不要:
- COCO class 15 (cat) と 16 (dog) を両方「pet」として扱う
- YOLO26m 単体でも dog 64% として猫を検出できている
- pet_id はカラー分析 (UV scatter) で決定（既存フロー）

## 環境構築手順

### 1. axmodel ダウンロード

```bash
python3 -c "
from huggingface_hub import hf_hub_download
hf_hub_download('AXERA-TECH/yolo26', 'ax650/yolo26l.axmodel', local_dir='/home/admin-user/models/yolo26')
hf_hub_download('AXERA-TECH/YOLO11', 'ax650/yolo11s.axmodel', local_dir='/home/admin-user/models/yolo11')
"
```

### 2. ax-samples ビルド

```bash
sudo apt install -y libopencv-dev  # 668MB, eMMC 空き要確認

cd /tmp
git clone --depth 1 https://github.com/AXERA-TECH/ax-samples.git
cd ax-samples

# 閾値を 0.10 に変更 (デフォルト 0.45 では暗所で検出不可)
sed -i 's/PROB_THRESHOLD = 0.45f/PROB_THRESHOLD = 0.10f/' \
  examples/ax650/ax_yolo26_steps.cc \
  examples/ax650/ax_yolo11_steps.cc

mkdir build && cd build
cmake -DBSP_MSP_DIR=/tmp/ax-llm-bsp/msp_3.6.2/out/ -DAXERA_TARGET_CHIP=ax650 ..
make ax_yolo26 ax_yolo11 -j6
```

### 3. 推論実行

```bash
# root 権限必要 (/dev/mem アクセス)
sudo ./ax_yolo26 \
  -m /home/admin-user/models/yolo26/ax650/yolo26l.axmodel \
  -i /path/to/panel.jpg

sudo ./ax_yolo11 \
  -m /home/admin-user/models/yolo11/ax650/yolo11s.axmodel \
  -i /path/to/panel.jpg
```

### 注意事項

- NPU アクセスに **root 権限が必要** (`/dev/mem` 経由)
- BSP SDK ヘッダーは `/tmp/ax-llm-bsp/msp_3.6.2/out/include/` にある
- eMMC 残量注意: OpenCV + ビルド環境で ~1GB 消費
- VLM (axllm serve) と NPU を時分割する必要あり（同時実行不可の可能性）
