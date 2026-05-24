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

## 実装フェーズ

### フェーズ1: keymap.c / keymap.h

依存関係がなく最初に実装できる。単体テストしやすい。

#### keymap.h

```c
#pragma once
#include <stdint.h>

typedef struct {
    uint8_t modifier;
    uint8_t reserved;
    uint8_t keycode[6];
} hid_keyboard_report_t;

// US HIDレポートをJIS HIDレポートに変換する
// 変換テーブルに載っていないキーはそのまま通過する
void translate_us_to_jis(const hid_keyboard_report_t *us_report,
                          hid_keyboard_report_t *jis_report);
```

#### keymap.c の実装方針

1. **変換エントリ構造体**をフラッシュ配置(`const`)で定義:

```c
typedef struct {
    uint8_t us_keycode;
    bool    us_shift;     // Shiftあり入力か
    uint8_t jis_keycode;
    int8_t  shift_delta;  // -1: Shift除去, 0: 変化なし, +1: Shift付加
} keymap_entry_t;
```

2. `translate_us_to_jis()` のロジック:
   - HIDレポートのkeycode[0..5]を順に走査
   - Shift状態（`modifier & 0x22`）を確認
   - 変換テーブルを線形探索（24エントリ程度なので十分高速）
   - マッチしたエントリのjis_keycodeとshift_deltaを適用
   - 変換後のmodifierはShift付加/除去を反映して更新
   - `reserved`は0x00のまま

3. **Alt+バッククォート**の特殊処理:
   - keycode == 0x35 かつ Alt（`modifier & 0x44`）のとき
   - keycodeはそのまま（0x35 = JIS 全角/半角）、Altビットをクリア

#### 注意事項

モディファイアは全キーコードで共有される1バイト。同時押しで「Shift除去が必要なキー」と
「Shift付加が必要なキー」が混在した場合は正しく変換できない（構造的制限）。
詳細は [keymap.md](keymap.md) の同時押しセクションを参照。

---

### フェーズ2: usb_host.c / usb_host.h

Pico-PIO-USBラッパー。コア1で動作する。

#### usb_host.h

```c
#pragma once

// コア1エントリポイント（multicore_launch_core1() に渡す）
void core1_entry(void);
```

#### usb_host.c の実装方針

1. **`core1_entry()`**:
   ```c
   void core1_entry(void) {
       // Pico-PIO-USB ホスト初期化
       pio_usb_configuration_t config = PIO_USB_DEFAULT_CONFIG;
       config.pin_dp = PIO_USB_DP_PIN;  // GPIO 0
       tuh_configure(1, TUH_CFGID_RPI_PIO_USB_CONFIGURATION, &config);
       tuh_init(1);

       while (true) {
           tuh_task();  // USBホストタスク（ポーリング）
       }
   }
   ```

2. **`tuh_hid_report_received_cb()`** コールバック（TinyUSBホスト側）:
   ```c
   void tuh_hid_report_received_cb(uint8_t dev_addr, uint8_t instance,
                                    uint8_t const *report, uint16_t len) {
       if (len < 8) return;

       hid_keyboard_report_t us_report, jis_report;
       memcpy(&us_report, report, sizeof(us_report));
       translate_us_to_jis(&us_report, &jis_report);

       // コア0へ転送（2ワード）
       multicore_fifo_push_blocking(
           ((uint32_t)jis_report.modifier   << 24) |
           ((uint32_t)jis_report.reserved   << 16) |
           ((uint32_t)jis_report.keycode[0] <<  8) |
            (uint32_t)jis_report.keycode[1]);
       multicore_fifo_push_blocking(
           ((uint32_t)jis_report.keycode[2] << 24) |
           ((uint32_t)jis_report.keycode[3] << 16) |
           ((uint32_t)jis_report.keycode[4] <<  8) |
            (uint32_t)jis_report.keycode[5]);

       // 次のレポートを要求
       tuh_hid_receive_report(dev_addr, instance);
   }
   ```

3. **`tuh_hid_mount_cb()`** でHIDレポート受信を開始:
   ```c
   void tuh_hid_mount_cb(uint8_t dev_addr, uint8_t instance,
                          uint8_t const *desc_report, uint16_t desc_len) {
       tuh_hid_receive_report(dev_addr, instance);
   }
   ```

---

### フェーズ3: usb_device.c / usb_device.h

TinyUSBデバイス側。コア0で動作する。

#### usb_device.h

```c
#pragma once
#include "keymap.h"

void usb_device_init(void);
// JISキーボードとしてHIDレポートを送信（busy時はドロップ）
void usb_device_send_report(const hid_keyboard_report_t *report);
```

#### usb_device.c の実装方針

1. **HIDディスクリプタ**をJISキーボードとして定義:
   - Usage Page: 0x01（Generic Desktop）
   - Usage: 0x06（Keyboard）
   - Boot Protocol Keyboard ディスクリプタを使用（8バイトレポート）

2. **`usb_device_init()`**: `tusb_init()` を呼び出す

3. **`usb_device_send_report()`**:
   ```c
   void usb_device_send_report(const hid_keyboard_report_t *report) {
       if (!tud_hid_ready()) return;  // busyならドロップ
       tud_hid_keyboard_report(0, report->modifier, report->keycode);
   }
   ```

4. **必須コールバック**:
   - `tud_hid_get_report_cb()`: 現在のレポートを返す
   - `tud_hid_set_report_cb()`: LED状態等（NumLock等）を受信
   - `tud_descriptor_device_cb()`: デバイスディスクリプタ
   - `tud_descriptor_configuration_cb()`: 設定ディスクリプタ
   - `tud_descriptor_string_cb()`: 文字列ディスクリプタ（製品名等）

5. **tusb_config.h** で設定:
   ```c
   #define CFG_TUD_HID  1
   #define CFG_TUD_ENDPOINT0_SIZE 64
   ```

---

### フェーズ4: keyforge-pico.c リライト

LEDブリンクコードを削除し、メインロジックを実装する。

#### main() の構成

```c
int main(void) {
    // 1. UART1初期化（GPIO 8/9、デバッグ用）
    uart_init(uart1, 115200);
    gpio_set_function(8, GPIO_FUNC_UART);
    gpio_set_function(9, GPIO_FUNC_UART);
    stdio_uart_init();  // または stdio_init_all() + UART設定

    // 2. TinyUSBデバイス初期化（コア0で行う）
    usb_device_init();

    // 3. コア1起動（Pico-PIO-USBホストタスク）
    multicore_launch_core1(core1_entry);

    // 4. コア0メインループ
    while (true) {
        tud_task();  // TinyUSBデバイスタスク

        // FIFOにデータがあれば読み取って送信
        if (multicore_fifo_rvalid()) {
            uint32_t w0 = multicore_fifo_pop_blocking();
            uint32_t w1 = multicore_fifo_pop_blocking();

            hid_keyboard_report_t report;
            report.modifier   = (w0 >> 24) & 0xFF;
            report.reserved   = 0x00;
            report.keycode[0] = (w0 >>  8) & 0xFF;
            report.keycode[1] =  w0        & 0xFF;
            report.keycode[2] = (w1 >> 24) & 0xFF;
            report.keycode[3] = (w1 >> 16) & 0xFF;
            report.keycode[4] = (w1 >>  8) & 0xFF;
            report.keycode[5] =  w1        & 0xFF;

            usb_device_send_report(&report);
        }
    }
}
```

---

### フェーズ5: CMakeLists.txt 更新

#### 追加すべき内容

1. **Pico-PIO-USB の取得**（FetchContent を推奨）:
   ```cmake
   include(FetchContent)
   FetchContent_Declare(
       pico_pio_usb
       GIT_REPOSITORY https://github.com/sekigon-gonnoc/Pico-PIO-USB.git
       GIT_TAG main
   )
   FetchContent_MakeAvailable(pico_pio_usb)
   ```

2. **ソースファイルの追加**:
   ```cmake
   add_executable(keyforge-pico
       src/keyforge-pico.c
       src/usb_host.c
       src/usb_device.c
       src/keymap.c
   )
   ```

3. **リンクライブラリの追加**:
   ```cmake
   target_link_libraries(keyforge-pico
       pico_stdlib
       pico_multicore
       tinyusb_device
       tinyusb_board
       pico_pio_usb
   )
   ```

4. **UART stdio 有効化**（USB stdioは無効のまま）:
   ```cmake
   pico_enable_stdio_uart(keyforge-pico 1)
   pico_enable_stdio_usb(keyforge-pico 0)
   ```

5. **インクルードパスを src/ に設定**（tusb_config.h 等も src/ に置く）:
   ```cmake
   target_include_directories(keyforge-pico PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
   ```

---

### フェーズ6: 動作確認

#### ビルド確認

```bash
cd build && cmake .. -G Ninja
ninja -C build
# エラーなく build/keyforge-pico.uf2 が生成されること
```

#### フラッシュ・基本動作確認

1. BOOTSELボタンを押しながらPicossciをPCに接続
2. `picotool load build/keyforge-pico.uf2 -fx` でフラッシュ
3. UARTモニタを接続（UART1, 115200bps, GPIO 8/9）
4. USキーボードをUSB-Aポートに接続
5. PCにJISキーボードとして認識されることを確認

#### キー変換確認チェックリスト

PCのキーボード設定をJISにした状態でテキストエディタに入力:

- [ ] `=` キー → `=` が入力される
- [ ] `[` キー → `[` が入力される
- [ ] `]` キー → `]` が入力される
- [ ] `\` キー → `¥` が入力される
- [ ] `'` キー → `'` が入力される
- [ ] `` ` `` キー → `` ` `` が入力される
- [ ] Shift+`2` → `@` が入力される
- [ ] Shift+`6` → `^` が入力される
- [ ] Shift+`-` → `_` が入力される
- [ ] Shift+`=` → `+` が入力される
- [ ] Shift+`;` → `:` が入力される
- [ ] Shift+`'` → `"` が入力される
- [ ] Shift+`` ` `` → `~` が入力される
- [ ] Alt+`` ` `` → 全角/半角トグル

## 実装順序のまとめ

```
1. keymap.c / keymap.h     ← 依存なし、単体テスト可能
2. usb_host.c / usb_host.h ← keymap.h に依存
3. usb_device.c / .h       ← TinyUSB のみ依存
4. keyforge-pico.c         ← すべてに依存
5. CMakeLists.txt 更新     ← ビルド通しながら並行して進める
```
