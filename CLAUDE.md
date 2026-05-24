# keyforge-pico

## プロジェクト概要

US配列キーボード → JIS配列キーボード変換アダプタ。

**背景**: Windowsはキーボードごとにレイアウト（US/JIS）を設定できず、システム全体で統一される。
そのため、USキーボードをJIS設定のWindowsに接続すると記号キーがすべてずれてしまう。
このアダプタはUSキーボードのHIDレポートをJIS配列に変換してPCへ送出することで、
Windowsの設定を変えずにUSキーボードを正しく使えるようにする。

- ハード: Picossci USBホスト（RP2040搭載）
  - 製品ページ: https://www.switch-science.com/products/9158
- USB-A側（PIO）: USキーボードをUSBホストとして受信
- USB Type-C側: PCへJIS HIDキーボードとして認識させる

## 使用ライブラリ

- Pico SDK 2.2.0 (`~/.pico-sdk/sdk/2.2.0/`)
- Pico-PIO-USB: PIOでUSBホスト実装（USB-A側）
  - https://github.com/sekigon-gonnoc/Pico-PIO-USB
- TinyUSB: USBデバイス/HID実装（USB Type-C側、Pico SDK同梱）
  - https://github.com/hathach/tinyusb

## ビルド方法

```bash
# 初回のみ: buildディレクトリ作成
mkdir -p build

# 初回 or CMakeLists.txt変更後
cd build && cmake .. -G Ninja

# 通常ビルド（プロジェクトルートから実行）
ninja -C build
```

生成物: `build/keyforge-pico.uf2`

## フラッシュ方法

```bash
# BOOTSELボタンを押しながら接続後
picotool load build/keyforge-pico.uf2 -fx

# OpenOCD (CMSIS-DAP接続時)
openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg \
  -c "adapter speed 5000; program build/keyforge-pico.elf verify reset exit"
```

## アーキテクチャ

→ [docs/architecture.md](docs/architecture.md)

## キー変換の方針

→ [docs/keymap.md](docs/keymap.md)


## 実装計画

→ [docs/implementation-plan.md](docs/implementation-plan.md)

## 開発上の制約

- RP2040のSRAM: 264KB。キー変換テーブルはフラッシュ配置（`const` で定義）
- USB HIDレポートのポーリング間隔: 1ms以内を目標
- stdio は UART のみ使用（`pico_enable_stdio_usb` は 0 のまま。USB Type-C をデバイスとして使うため）
- UART は UART1 を使用（GPIO 0/1 が PIO USB で占有されているため）。TX=GPIO 8, RX=GPIO 9
