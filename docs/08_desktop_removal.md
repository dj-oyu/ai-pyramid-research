# デスクトップ環境の削除と復旧

## 削除前の状態 (2026-03-24)

- **デスクトップ**: Xfce4 (lxdmディスプレイマネージャ)
- **Xサーバー**: Xorg `:0` (tty7)
- **リモートデスクトップ**: xorgxrdp
- **Snap**: Firefox, GNOME libs, cups, mesa等 (5.8GB)
- **Docker**: docker-ce + containerd (353MB, コンテナなし)
- **HDMI出力**: Xfceデスクトップ表示中

## 削除しても影響がないもの

- SSH接続
- StackFlow推論 (llm_sys, llm_llm, llm_vlm, OpenAI API)
- ec_cli ハードウェア制御
- 3DGS レンダラー (AX_VO API / /dev/fb0 直接アクセス、Xorg不要)
- Tailscale VPN

## 削除コマンド

### Step 1: snap 全削除 (~5.8GB 解放)

```bash
# snapパッケージを依存順に削除
sudo snap remove --purge firefox
sudo snap remove --purge cups
sudo snap remove --purge gnome-46-2404
sudo snap remove --purge gnome-42-2204
sudo snap remove --purge gtk-common-themes
sudo snap remove --purge mesa-2404
sudo snap remove --purge core24
sudo snap remove --purge core22
sudo snap remove --purge bare
sudo snap remove --purge snapd

# snapdをaptから完全削除
sudo apt purge -y snapd
sudo rm -rf /snap /var/snap /var/lib/snapd /var/cache/snapd

# snapdの再インストールを防止
sudo tee /etc/apt/preferences.d/no-snap.pref <<'PREF'
Package: snapd
Pin: release a=*
Pin-Priority: -10
PREF
```

### Step 2: Docker 削除 (~353MB 解放)

```bash
sudo systemctl stop docker containerd
sudo apt purge -y docker-ce docker-ce-cli docker-buildx-plugin \
  docker-ce-rootless-extras docker-compose-plugin containerd.io
sudo rm -rf /var/lib/docker /var/lib/containerd
```

### Step 3: デスクトップ環境削除 (~500MB+ 解放)

```bash
# ディスプレイマネージャを停止
sudo systemctl stop lxdm
sudo systemctl disable lxdm

# Xfce4 + 関連パッケージを削除
sudo apt purge -y \
  xfce4 xfce4-* xfwm4 xfdesktop4 xfdesktop4-data xfconf thunar \
  elementary-xfce-icon-theme \
  lxdm \
  xorg xserver-xorg xserver-xorg-core xserver-xorg-input-all \
  xserver-xorg-video-all xserver-xorg-legacy xserver-common \
  xorgxrdp \
  x11-apps x11-utils x11-xserver-utils x11-xkb-utils x11-common \
  x11-session-utils xfonts-base xfonts-scalable xfonts-encodings xfonts-utils \
  pulseaudio pulseaudio-utils pavucontrol \
  desktop-base \
  adwaita-icon-theme humanity-icon-theme tango-icon-theme \
  greybird-gtk-theme \
  python3-pyqt5 python3-pyqt5.sip \
  libwebkit2gtk-4.0-37 libjavascriptcoregtk-4.0-18 \
  mesa-vulkan-drivers libgl1-mesa-dri libgl1-amber-dri libllvm15 \
  cups-browsed cups-common

# 依存パッケージを自動削除
sudo apt autoremove -y
sudo apt clean
```

### Step 4: 確認

```bash
df -h /
# 6GB以上空くはず
```

## 復旧コマンド (Xfce4)

元の環境に戻す場合:

```bash
# Xfce4 + lxdm + Xorgを再インストール
sudo apt update
sudo apt install -y xfce4 xfce4-power-manager lxdm xorg \
  elementary-xfce-icon-theme greybird-gtk-theme \
  pulseaudio pulseaudio-utils \
  fonts-wqy-zenhei fonts-ubuntu

# lxdmを有効化して起動
sudo systemctl enable lxdm
sudo systemctl start lxdm
```

## 代替: Wayland + Weston (軽量GUI)

Xfce復旧ではなく、より軽量なWayland環境を構築する場合:

```bash
# Weston (Waylandリファレンスコンポジタ) をインストール
sudo apt update
sudo apt install -y weston

# 起動 (HDMI出力)
# ttyからログインして実行:
weston --backend=drm-backend.so
```

Westonの特徴:
- Xorg + Xfce に比べて大幅に軽量 (~数十MB)
- DRMバックエンドでHDMI直接出力
- ターミナルエミュレータ内蔵 (`weston-terminal`)
- Xwayland を追加すればX11アプリも動作

### Weston 自動起動の設定

```bash
# systemdサービスとして登録
sudo tee /etc/systemd/system/weston.service <<'SERVICE'
[Unit]
Description=Weston Wayland Compositor
After=multi-user.target

[Service]
Type=simple
User=root
Environment=XDG_RUNTIME_DIR=/run/user/0
ExecStart=/usr/bin/weston --backend=drm-backend.so --tty=7
Restart=on-failure

[Install]
WantedBy=multi-user.target
SERVICE

sudo systemctl enable weston
sudo systemctl start weston
```

### Cage (キオスクモード Waylandコンポジタ)

単一アプリ表示のみの超軽量コンポジタ。3DGSビューワー等の全画面表示に最適:

```bash
sudo apt install -y cage
# 単一アプリを全画面で起動:
cage -- weston-terminal
```
