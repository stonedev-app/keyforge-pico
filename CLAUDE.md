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

### CPUコア分担（Pico-PIO-USBの推奨構成）

- **コア0**: TinyUSB デバイスタスク（PC側 HID送信）
- **コア1**: Pico-PIO-USB ホストタスク（USキーボード受信）

### 処理フロー

```
[USキーボード] --USB-A(PIO)--> [コア1: ホスト受信]
                                      |
                               USキーコード + モディファイア
                                      |
                               [キー変換テーブル]
                                      |
                               JISキーコード + モディファイア
                                      |
                               [コア0: HID送信] --USB Type-C--> [PC]
```

### ピン設定（Picossciボード）

- `PIO_USB_DP_PIN` = GPIO 0 (D+)
- `PIO_USB_DM_PIN` = GPIO 1 (D-)

Pico-PIO-USB の設定: `PIO_USB_DP_PIN` を 0 にすれば DM は自動的に 1 になる（連番前提）。

## キー変換の方針

参考実装: [m47ch4n/qmk-translate-ansi-to-jis](https://github.com/m47ch4n/qmk-translate-ansi-to-jis)
（QMKベースのANSI→JIS変換ライブラリ。変換テーブルとモディファイア処理のロジックが参考になる）

- HIDキーコードレベルで変換（スキャンコードではない）
- US→JISで記号の位置が異なるキーは変換テーブルで対応
- モディファイア変換が必要なケースあり（例: US `@` [Shift+2] → JIS `@` [単独キー]）

### 変換対象キー一覧

| US入力 | 出力文字 | 備考 |
|--------|---------|------|
| `=` | `=` | Shift状態変更（JIS: Shift+-） |
| `[` | `[` | キーコード変更（JIS: `]`位置） |
| `\` | `¥` | キーコード変更（JIS: INT3） |
| `]` | `]` | キーコード変更（JIS: NUHSキー） |
| `'` | `'` | キーコード変更（JIS: Shift+7） |
| `` ` `` | `` ` `` | キーコード変更（JIS: Shift+`@`） |
| Shift+2（`@`） | `@` | モディファイア除去（JIS: `[`位置） |
| Shift+6（`^`） | `^` | キーコード変更（JIS: `=`位置） |
| Shift+7（`&`） | `&` | キーコード変更（JIS: Shift+6） |
| Shift+8（`*`） | `*` | キーコード変更（JIS: Shift+`'`位置） |
| Shift+9（`(`） | `(` | キーコード変更（JIS: Shift+8） |
| Shift+0（`)`） | `)` | キーコード変更（JIS: Shift+9） |
| Shift+`=`（`+`） | `+` | キーコード変更（JIS: Shift+`;`） |
| Shift+`-`（`_`） | `_` | キーコード変更（JIS: Shift+INT1） |
| Shift+`[`（`{`） | `{` | キーコード変更（JIS: Shift+`]`位置） |
| Shift+`\`（`\|`） | `\|` | キーコード変更（JIS: Shift+¥） |
| Shift+`]`（`}`） | `}` | キーコード変更（JIS: Shift+NUHS） |
| Shift+`;`（`:`） | `:` | モディファイア除去（JIS: `'`位置） |
| Shift+`'`（`"`） | `"` | キーコード変更（JIS: Shift+2） |
| Shift+`` ` ``（`~`） | `~` | キーコード変更（JIS: Shift+`=`） |
| Alt+`` ` `` | 全角/半角 | JISキーコード（0x35）に置換 |

## 開発上の制約

- RP2040のSRAM: 264KB。キー変換テーブルはフラッシュ配置（`const` で定義）
- USB HIDレポートのポーリング間隔: 1ms以内を目標
- stdio は UART のみ使用（`pico_enable_stdio_usb` は 0 のまま。USB Type-C をデバイスとして使うため）
- コア間通信: `multicore_fifo` または共有変数 + メモリバリア

## ファイル構成（予定）

```
keyforge-pico.c      # main、コア0エントリ
usb_host.c/.h        # コア1、Pico-PIO-USBラッパー
usb_device.c/.h      # コア0、TinyUSB HIDデバイス
keymap.c/.h          # US→JIS変換テーブル
```
