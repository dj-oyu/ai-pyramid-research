# ec_cli ハードウェア制御コマンドリファレンス

AI Pyramid Pro の組み込みコントローラ(EC)を操作するCLIツール。
LED、ファン、電源、LCD、USB、PCIe、ボタン等のハードウェアを制御する。

## 基本情報

- バイナリ: `/usr/local/m5stack/bin/ec_cli-1.0`
- 通信先: `ec_proxy-1.0` デーモン（ZMQ Unixソケット経由）
- ec_proxyデーモン: `/usr/local/m5stack/bin/ec_proxy-1.0`
- ZMQソケット: `/tmp/rpc.ec_prox` (RPC), `/tmp/llm/ec_prox.event.socket` (イベント)
- **root権限が必要** (`sudo` で実行すること)

## 構文

```bash
sudo ec_cli-1.0 device <オプション>              # 読み取り
sudo ec_cli-1.0 device <オプション> -d '<値>'     # 書き込み
sudo ec_cli-1.0 exec -l                          # 低レベル関数一覧
sudo ec_cli-1.0 exec -f <関数名> -d '<JSON>'     # 低レベル関数呼び出し
sudo ec_cli-1.0 echo                             # イベントストリーム購読
```

## レスポンス形式

全コマンドのレスポンスはJSON:
```json
{
  "created": <unix_timestamp>,
  "data": <値>,
  "object": "",
  "request_id": "",
  "work_id": "<内部API名>"
}
```
`data` フィールドに結果が入る。型はコマンドにより int / string / object。

## コマンド一覧と実測値

### システム情報

```bash
# ECファームウェアバージョン
sudo ec_cli-1.0 device --version
# → data: 1
```

### ファン制御

```bash
# ファン回転速度(RPM)を取得
sudo ec_cli-1.0 device --fanspeed    # -F
# → data: 180  (RPM)

# ファンPWMデューティ比を取得/設定 (0-100)
sudo ec_cli-1.0 device --fan         # -f  読み取り
# → data: 40  (40%)
sudo ec_cli-1.0 device --fan -d '80' # PWMを80%に設定
```

### 電源・消費電力

```bash
# ボード消費電力の詳細を取得
sudo ec_cli-1.0 device --board       # -B
# → data: {
#     "pcie0_mv": 3344, "pcie0_ma": 0,     ← PCIeスロット0 (3.3V, 未使用)
#     "pcie1_mv": 3320, "pcie1_ma": 0,     ← PCIeスロット1 (3.3V, 未使用)
#     "usb1_mv": 5056,  "usb1_ma": 4,      ← USBポート1 (5V, 4mA)
#     "usb2_mv": 5048,  "usb2_ma": 0,      ← USBポート2 (5V, 未使用)
#     "INVDD_mv": 15048, "INVDD_ma": 372,  ← 内部電源 (15V, 372mA ≈ 5.6W)
#     "EXTVDD_mv": 15048, "EXTVDD_ma": 32  ← 外部電源 (15V, 32mA)
#   }

# USB PD給電情報
sudo ec_cli-1.0 device --pd_power_info
# → data: {"voltage": "15 V", "current": "1.25 A"}  (≈18.75W)

# CPUコア電圧
sudo ec_cli-1.0 device --vddcpu      # -c
# → data: 20

# 電源レール正常性チェック (1=正常, 0=異常)
sudo ec_cli-1.0 device --V3_3_good   # → data: 1
sudo ec_cli-1.0 device --V1_8_good   # → data: 1

# 外部電源出力の制御
sudo ec_cli-1.0 device --ext_power   # -p

# メインボード電源スイッチ (注意: 電源が切れる)
sudo ec_cli-1.0 device --board_power # -b

# システムシャットダウン (注意: 電源が切れる)
sudo ec_cli-1.0 device --poweroff

# スケジュール電源ON/OFF (単位: 秒? 0=無効)
sudo ec_cli-1.0 device --poweron_time   # → data: 0 (無効)
sudo ec_cli-1.0 device --poweroff_time  # → data: 30000
```

### RGB LED制御

```bash
# LED表示モード (1=有効)
sudo ec_cli-1.0 device --rgb         # -r
# → data: 1

# LED数
sudo ec_cli-1.0 device --rgb_size
# → data: 48  (48個のRGB LED)

# 個別LEDの色を取得
sudo ec_cli-1.0 device --rgb_get_color -d '{"rgb_index":0}'

# 個別LEDの色を設定 (rgb_color: 24bit = R<<16 | G<<8 | B)
sudo ec_cli-1.0 device --rgb_set_color -d '{"rgb_index":0,"rgb_color":16711680}'  # 赤 0xFF0000
sudo ec_cli-1.0 device --rgb_set_color -d '{"rgb_index":0,"rgb_color":65280}'     # 緑 0x00FF00
sudo ec_cli-1.0 device --rgb_set_color -d '{"rgb_index":0,"rgb_color":255}'       # 青 0x0000FF
```

### LCD制御

```bash
# LCD表示モード
sudo ec_cli-1.0 device --lcd         # -l
# → data: 2

# バックライト輝度 (0-255)
sudo ec_cli-1.0 device --lcd_brightness
# → data: 207

# LCDに文字列を出力
sudo ec_cli-1.0 device --lcd_putc -d 'Hello'  # -P

# LCD RAMバッファに直接書き込み
sudo ec_cli-1.0 device --lcd_ram -d '<data>'
```

### ボタン状態

```bash
# ボタン押下状態 (0=離した, 1=押した)
sudo ec_cli-1.0 device --head_button  # → data: 0
sudo ec_cli-1.0 device --lcd_button   # → data: 0

# ボタンイベントハンドラの取得/設定
sudo ec_cli-1.0 device --ec_button_head_event
sudo ec_cli-1.0 device --soc_button_head_event
sudo ec_cli-1.0 device --ec_button_lcd_event
```

### USB制御

```bash
# USBダウンストリームポートのON/OFF
sudo ec_cli-1.0 device --usbds1      # ポート1
sudo ec_cli-1.0 device --usbds2      # ポート2
sudo ec_cli-1.0 device --usbds3      # ポート3

# USB大電力モード
sudo ec_cli-1.0 device --usbds1_big
sudo ec_cli-1.0 device --usbds2_big

# USBハブリセット
sudo ec_cli-1.0 device --gl3510_reset
```

### PCIe制御

```bash
# PCIeスロットのON/OFF
sudo ec_cli-1.0 device --pcie0
sudo ec_cli-1.0 device --pcie1

# PCIeデバイスの有無チェック (0=なし, 1=あり)
sudo ec_cli-1.0 device --pcie0_exists  # → data: 0 (未装着)
sudo ec_cli-1.0 device --pcie1_exists  # → data: 0 (未装着)
```

### HDMI

```bash
# HDMI OUT入力ソース切替 (IN ↔ AX8850)
sudo ec_cli-1.0 device --hdmi_loop_en
```

### ネットワーク (EC経由のIP取得)

```bash
sudo ec_cli-1.0 device --ip_eth0   # → data: "x.x.x.x" (バイト順が逆になる場合あり)
sudo ec_cli-1.0 device --ip_eth1
sudo ec_cli-1.0 device --ip_wlan   # → data: "0.0.0.0" (未接続)
```

> 注: ECが返すIPのバイト順が逆になっている場合がある。`ip addr` コマンドの方が正確。

### Groveインターフェース

```bash
sudo ec_cli-1.0 device --grove_uart  # UART ON/OFF
sudo ec_cli-1.0 device --grove_iic   # I2C ON/OFF
```

### I2C直接アクセス

```bash
sudo ec_cli-1.0 device --i2c_get_reg -d '<params>'
sudo ec_cli-1.0 device --i2c_set_reg -d '<params>'
```

### プロキシ制御

```bash
# 自動制御モード (1=有効)
sudo ec_cli-1.0 device --fun_auto
# → data: 1

# Modbus通信ボーレート
sudo ec_cli-1.0 device --modbus_speed
```

### Modbusレジスタ（低レベル）

```bash
sudo ec_cli-1.0 device --ec_modbus_set_bit -d '<params>'
sudo ec_cli-1.0 device --ec_modbus_get_bit -d '<params>'
sudo ec_cli-1.0 device --ec_modbus_input_bits
sudo ec_cli-1.0 device --ec_modbus_input_registers
sudo ec_cli-1.0 device --ec_modbus_get_hold_registers -d '<params>'
sudo ec_cli-1.0 device --ec_modbus_set_hold_registers -d '<params>'
```

### 永続化

```bash
# 現在の設定をフラッシュに保存 (再起動後も有効)
sudo ec_cli-1.0 device --flash_switch  # スイッチ設定 (coilレジスタ 4-14)
sudo ec_cli-1.0 device --flash_value   # 値設定 (holdingレジスタ 1,2,11,12,14,16-21)
```

## 現在のデバイス状態サマリー (2026-03-24 計測)

| 項目 | 値 | 補足 |
|------|-----|------|
| ECファームウェア | v1 | |
| USB PD給電 | 15V / 1.25A (18.75W) | |
| 内部電源消費 | 15V / 372mA (5.6W) | INVDD |
| ファン | 180 RPM / PWM 40% | |
| RGB LED | 48個, モード1 | |
| LCD輝度 | 207/255 | |
| 3.3V / 1.8V レール | 両方正常 | |
| PCIeスロット | 0/1 両方未装着 | M.2 SSD増設可 |
| USBポート1 | 5.056V / 4mA | 接続デバイスあり |
| USBポート2 | 5.048V / 未使用 | |
| 自動制御モード | 有効 | |

## 注意事項

- **root権限が必要**: ec_proxyのZMQソケット (`/tmp/rpc.ec_prox`) へのアクセスにroot権限が必要
- **破壊的コマンド**: `--poweroff`, `--board_power` は実際にデバイスの電源を切る
- **永続化**: `--flash_switch` / `--flash_value` でフラッシュに書き込むと再起動後も有効
- **IP表示**: ECが返すIPアドレスのバイト順が逆になる場合がある
- ec_proxy停止時は全コマンドが `call ... faile!` で失敗する
