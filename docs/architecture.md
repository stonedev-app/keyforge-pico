# システムアーキテクチャ

## 概要

keyforge-picoは、USレイアウトキーボードからのHID入力をJISレイアウトに変換して
PCへ送出するRaspberry Pi Pico（RP2040）ベースのUSBアダプタ。

## ハードウェア構成

| 項目 | 内容 |
|------|------|
| ボード | Picossci USBホスト（RP2040搭載） |
| USB入力 | USB-A（PIOによるUSBホスト）← USキーボード |
| USB出力 | USB Type-C（TinyUSBデバイス）→ PC |
| デバッグ | UART1（GPIO 8: TX、GPIO 9: RX） |

## GPIO割り当て

| GPIO | 用途 |
|------|------|
| GPIO 0 | PIO USB D+（`PIO_USB_DP_PIN`） |
| GPIO 1 | PIO USB D−（連番により自動割り当て） |
| GPIO 8 | UART1 TX（デバッグ） |
| GPIO 9 | UART1 RX（デバッグ） |

GPIO 0/1がPIO USBに占有されるため、デバッグstdioはUART1を使用する。
`pico_enable_stdio_usb`は0固定（USB Type-CをHIDデバイスとして使用するため）。

## デュアルコア分担

```
コア0: TinyUSB デバイスタスク（PC側 HID送信）
コア1: Pico-PIO-USB ホストタスク（USキーボード受信・変換）
```

この分担はPico-PIO-USBの推奨構成に従う。コア1で受信・変換したHIDレポートは
`multicore_fifo`でコア0へ転送し、コア0がTinyUSBを通じてPCへ送出する。

## HIDレポート構造

USBキーボードHIDレポートは8バイト固定:

| バイト | 内容 |
|--------|------|
| 0 | モディファイア（8-bit） |
| 1 | 予約（0x00） |
| 2〜7 | キーコード（最大6キー同時押し、各8-bit） |

### モディファイアビット配置

| bit | 意味 |
|-----|------|
| 0 | 左 Ctrl |
| 1 | 左 Shift |
| 2 | 左 Alt |
| 3 | 左 GUI |
| 4 | 右 Ctrl |
| 5 | 右 Shift |
| 6 | 右 Alt |
| 7 | 右 GUI |

Shift判定: `modifier & 0x22`（bit1: 左Shift、bit5: 右Shift）
Alt判定: `modifier & 0x44`（bit2: 左Alt、bit6: 右Alt）

## コア間通信

`multicore_fifo`を使用。HIDレポート8バイトを32-bit × 2ワードにパッキングして転送する。

### エンコード仕様

```c
// コア1: 送信（translate_us_to_jis 後のJISレポートを送る）
multicore_fifo_push_blocking(
    ((uint32_t)report.modifier   << 24) |
    ((uint32_t)report.reserved   << 16) |
    ((uint32_t)report.keycode[0] <<  8) |
     (uint32_t)report.keycode[1]);
multicore_fifo_push_blocking(
    ((uint32_t)report.keycode[2] << 24) |
    ((uint32_t)report.keycode[3] << 16) |
    ((uint32_t)report.keycode[4] <<  8) |
     (uint32_t)report.keycode[5]);

// コア0: 受信
uint32_t w0 = multicore_fifo_pop_blocking();
uint32_t w1 = multicore_fifo_pop_blocking();
uint8_t modifier   = (w0 >> 24) & 0xFF;
uint8_t keycode[6] = {
    (w0 >>  8) & 0xFF, w0         & 0xFF,
    (w1 >> 24) & 0xFF, (w1 >> 16) & 0xFF,
    (w1 >>  8) & 0xFF,  w1        & 0xFF,
};
```

FIFOが満杯のとき`push_blocking()`はコア1をブロックする。
コア0が継続的にpopするため実用上は問題ないが、USBホストのポーリングタイミングに
影響する可能性がある（既知の制約）。

## 処理フロー

```
[USキーボード]
    │ USB-A (PIO, GPIO 0/1)
    ↓
[コア1: tuh_hid_report_received_cb()]
    │ translate_us_to_jis()
    ↓
[JIS HIDレポート]
    │ multicore_fifo_push_blocking() × 2
    ↓
[コア0: FIFOポーリングループ]
    │ usb_device_send_report()
    ↓
[PC] ← USB Type-C (TinyUSB HIDデバイス)
```

## 外部ライブラリ

| ライブラリ | 用途 | 取得方法 |
|-----------|------|---------|
| Pico SDK 2.2.0 | ベースSDK | `~/.pico-sdk/sdk/2.2.0/` |
| Pico-PIO-USB | PIOによるUSBホスト実装 | CMake FetchContent または git submodule |
| TinyUSB | USBデバイス/HID実装 | Pico SDK同梱 |
