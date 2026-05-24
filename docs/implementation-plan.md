# 実装計画

## 現状

`keyforge-pico.c`はLEDブリンクのスケルトンのみ。実装すべきファイルはすべて未作成。

## モジュール構成

```
src/
  keyforge-pico.c    main()・コア0メインループ
  usb_host.c / .h    コア1エントリ・Pico-PIO-USBラッパー
  usb_device.c / .h  TinyUSB HIDデバイス初期化・レポート送信
  keymap.c / .h      US→JIS変換テーブル・変換ロジック
```

### 依存関係

```
keyforge-pico.c
  ├── usb_host.h        (core1_entry 宣言)
  └── usb_device.h      (usb_device_init / usb_device_send_report)

usb_host.c
  └── keymap.h          (translate_us_to_jis)
      └── multicore_fifo_push_blocking() でコア0へ送信

usb_device.c
  └── TinyUSB HIDコールバック実装（tusb_config.h に依存）

keymap.c
  └── 変換テーブル（const定義、フラッシュ配置）
```

## 実装ステップ

動作確認しながら積み上げる6ステップ構成。各ステップは単独でビルド・フラッシュして確認する。

---

### ステップ1: マルチコア基礎

**目標**: コア1で生成した値がコア0のprintに表示される

- `src/keyforge-pico.c` に `core1_entry()` を追加
- コア1: カウンタをインクリメントして `multicore_fifo_push_blocking()` で送信
- コア0: `multicore_fifo_rvalid()` で確認後 `pop_blocking()` して `printf()`
- CMakeLists.txt: `pico_multicore` をリンク、`pico_enable_stdio_uart` を 1 に

参照: [architecture.md](architecture.md) のコア間通信セクション

---

### ステップ2: TinyUSBデバイス単体

**目標**: PCがキーボードとして認識し、タイマーで10秒ごとにAが入力される

- `src/tusb_config.h` 作成（`CFG_TUD_HID 1`）
- `src/usb_device.c / usb_device.h` 実装
  - HIDディスクリプタ（Boot Protocol Keyboard）
  - `usb_device_init()` / `usb_device_send_report()`
  - 必須コールバック（`tud_descriptor_device_cb` 等）
- `src/keyforge-pico.c`: `tud_task()` ループ + タイマーで定期送信
- CMakeLists.txt: `tinyusb_device` / `tinyusb_board` をリンク

コア1起動は不要（コア0のみ）。

---

### ステップ3: Pico-PIO-USBホスト

**目標**: USBキーボードの入力がUARTに出力される

- `src/usb_host.c / usb_host.h` 実装
  - `core1_entry()`: `tuh_init()` → `tuh_task()` ループ
  - `tuh_hid_mount_cb()`: `tuh_hid_receive_report()` でレポート受信開始
  - `tuh_hid_report_received_cb()`: レポート内容を `printf()` で出力
- CMakeLists.txt: FetchContent で `pico_pio_usb` を取得・リンク

```cmake
include(FetchContent)
FetchContent_Declare(
    pico_pio_usb
    GIT_REPOSITORY https://github.com/sekigon-gonnoc/Pico-PIO-USB.git
    GIT_TAG main
)
FetchContent_MakeAvailable(pico_pio_usb)
```

---

### ステップ4: コア連携・そのまま転送

**目標**: USキーボードの入力がPCにそのまま届く

- ステップ2（TinyUSBデバイス）+ ステップ3（Pico-PIO-USBホスト）を統合
- `usb_host.c`: レポートをそのままFIFOに push（変換なし）
- `keyforge-pico.c`: FIFOから受信 → `usb_device_send_report()`
- FIFOのエンコード仕様は [architecture.md](architecture.md) を参照

---

### ステップ5: 変換前後のprint確認

**目標**: `translate_us_to_jis()` の変換結果をUARTで目視確認できる

- `src/keymap.c / keymap.h` 実装
  - `hid_keyboard_report_t` 型定義
  - 変換エントリ構造体（`us_keycode` / `us_shift` / `jis_keycode` / `shift_delta`）
  - `translate_us_to_jis()` 実装（詳細は [keymap.md](keymap.md) 参照）
- `usb_host.c`: `translate_us_to_jis()` を呼び、変換前後を両方 `printf()`
- FIFOには変換後レポートを送る

---

### ステップ6: キー変換を本番に組み込み

**目標**: JIS変換が正しく動く

- printデバッグを削除（または残す）
- 全体通しで動作確認
- PCのキーボード設定をJISにした状態でテキストエディタに入力して検証:

| 入力 | 期待する出力 |
|------|------------|
| `=` | `=` |
| `[` | `[` |
| `]` | `]` |
| `\` | `¥` |
| `'` | `'` |
| `` ` `` | `` ` `` |
| Shift+`2` | `@` |
| Shift+`6` | `^` |
| Shift+`-` | `_` |
| Shift+`=` | `+` |
| Shift+`;` | `:` |
| Shift+`'` | `"` |
| Shift+`` ` `` | `~` |
| Alt+`` ` `` | 全角/半角トグル |

全キーの詳細は [keymap.md](keymap.md) を参照。
