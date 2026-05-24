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

### HID レポート構造

USB HID キーボードレポートは 8 バイト固定:

| バイト | 内容 |
|--------|------|
| 0 | モディファイア（8-bit） |
| 1 | 予約（0x00） |
| 2〜7 | キーコード（最大6キー同時押し、各 8-bit） |

モディファイアビット配置:
- bit0: 左 Ctrl / bit1: 左 Shift / bit2: 左 Alt / bit3: 左 GUI
- bit4: 右 Ctrl / bit5: 右 Shift / bit6: 右 Alt / bit7: 右 GUI

### コア間通信

`multicore_fifo` を使用。HID レポート 8 バイトを 32-bit × 2回に分けて送受信:

```c
// コア1 → コア0 へ送信
multicore_fifo_push_blocking(
    ((uint32_t)report.modifier << 24) | ((uint32_t)report.reserved << 16) |
    ((uint32_t)report.keycode[0] << 8) | report.keycode[1]);
multicore_fifo_push_blocking(
    ((uint32_t)report.keycode[2] << 24) | ((uint32_t)report.keycode[3] << 16) |
    ((uint32_t)report.keycode[4] << 8)  | report.keycode[5]);

// コア0 側で受信
uint32_t w0 = multicore_fifo_pop_blocking();
uint32_t w1 = multicore_fifo_pop_blocking();
uint8_t modifier   = (w0 >> 24) & 0xFF;
uint8_t keycode[6] = {
    (w0 >> 8) & 0xFF, w0 & 0xFF,
    (w1 >> 24) & 0xFF, (w1 >> 16) & 0xFF,
    (w1 >> 8) & 0xFF,  w1 & 0xFF
};
```

> **備考**: FIFO が満杯のとき `push_blocking()` はコア1をブロックする。コア0が継続的に pop しているため実用上は問題ないが、USB ホストのポーリングタイミングに影響する可能性がある。

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
- UART は UART1 を使用（GPIO 0/1 が PIO USB で占有されているため）。TX=GPIO 8, RX=GPIO 9

## ファイル構成（予定）

```
keyforge-pico.c      # main、コア0エントリ
usb_host.c/.h        # コア1、Pico-PIO-USBラッパー
usb_device.c/.h      # コア0、TinyUSB HIDデバイス
keymap.c/.h          # US→JIS変換テーブル
```
