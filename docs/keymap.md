# US→JIS キーマップ仕様

## 基本方針

- HIDキーコードレベルで変換（スキャンコードではない）
- 変換テーブルに載っていないキーはそのまま通過（英字・数字・ファンクションキー等）
- Shiftキーの有無によって別エントリとして扱う
- モディファイアの除去・追加が必要なケースあり

## HIDキーコード対照表

### 変換対象キー（US入力 → JIS出力）

凡例:
- **US HID**: USキーボードから受信するHIDキーコード
- **US Shift**: そのときのShift状態（Yes = Shiftあり）
- **JIS HID**: PCへ送出するHIDキーコード
- **JIS Shift**: 送出時のShift状態（出力Shiftはモディファイア全体で管理）

| 出力文字 | US HID | US Shift | JIS HID | JIS Shift | 備考 |
|---------|--------|----------|---------|-----------|------|
| `=` | 0x2E (`=`) | No | 0x2D (`-`) | Yes | JIS `-/=` キーのShift側 |
| `[` | 0x2F (`[`) | No | 0x30 (`[`) | No | JIS `[/{` キー |
| `¥` | 0x31 (`\`) | No | 0x89 (INT3) | No | JIS ¥キー |
| `]` | 0x30 (`]`) | No | 0x32 (NUHS) | No | JIS `]/}` キー |
| `'` | 0x34 (`'`) | No | 0x24 (`7`) | Yes | JIS `7/'` キーのShift側 |
| `` ` `` | 0x35 (`` ` ``) | No | 0x2F (`@`) | Yes | JIS `@/`` ` `` ` キーのShift側 |
| `@` | 0x1F (`2`) | Yes | 0x2F (`@`) | No | Shiftを除去、JIS `@/`` ` `` ` キー |
| `^` | 0x23 (`6`) | Yes | 0x2E (`^`) | No | Shiftを除去、JIS `^/~` キー |
| `&` | 0x24 (`7`) | Yes | 0x23 (`6`) | Yes | JIS `6/&` キーのShift側 |
| `*` | 0x25 (`8`) | Yes | 0x34 (`:`) | Yes | JIS `:/＊` キーのShift側 |
| `(` | 0x26 (`9`) | Yes | 0x25 (`8`) | Yes | JIS `8/(` キーのShift側 |
| `)` | 0x27 (`0`) | Yes | 0x26 (`9`) | Yes | JIS `9/)` キーのShift側 |
| `+` | 0x2E (`=`) | Yes | 0x33 (`;`) | Yes | JIS `;/+` キーのShift側 |
| `_` | 0x2D (`-`) | Yes | 0x87 (INT1) | Yes | JIS INT1 (`_/ろ`) のShift側 |
| `{` | 0x2F (`[`) | Yes | 0x30 (`[`) | Yes | JIS `[/{` キーのShift側 |
| `\|` | 0x31 (`\`) | Yes | 0x89 (INT3) | Yes | JIS ¥/\| キーのShift側 |
| `}` | 0x30 (`]`) | Yes | 0x32 (NUHS) | Yes | JIS `]/}` キーのShift側 |
| `:` | 0x33 (`;`) | Yes | 0x34 (`:`) | No | Shiftを除去、JIS `:/＊` キー |
| `"` | 0x34 (`'`) | Yes | 0x1F (`2`) | Yes | JIS `2/"` キーのShift側 |
| `~` | 0x35 (`` ` ``) | Yes | 0x2E (`^`) | Yes | JIS `^/~` キーのShift側 |
| 全角/半角 | 0x35 (`` ` ``) | Alt | 0x35 (全角/半角) | No | Altを除去 |

### 変換不要キー（通過）

英字（0x04–0x1D）、数字（0x1E–0x27 のうち変換対象外）、
ファンクションキー、方向キー、Ctrl・Alt・GUI等は変換せずそのまま通過する。

## JIS固有キーコード

| キーコード | 意味 | 物理位置 |
|-----------|------|---------|
| 0x35 | 全角/半角 | `` ` `` の位置（キーボード左上） |
| 0x87 | INT1（`_/ろ`） | `/` の右（Enterの左下） |
| 0x89 | INT3（`¥/\|`） | `]` の右（Backspaceの左） |
| 0x32 | NUHS（`]/}`） | JIS `]` キー（US `\` の右上相当） |

## モディファイア操作の分類

### Shiftを除去するケース

US側でShiftが押されているが、JIS側ではShiftなしで目的の文字が出るケース:

| US入力 | 出力文字 | 操作 |
|--------|---------|------|
| Shift+`2` (`@`) | `@` | Shift除去 + keycode変更 |
| Shift+`6` (`^`) | `^` | Shift除去 + keycode変更 |
| Shift+`;` (`:`) | `:` | Shift除去 + keycode変更 |

### Shiftを付加するケース

US側でShiftなしだが、JIS側ではShiftありで目的の文字が出るケース:

| US入力 | 出力文字 | 操作 |
|--------|---------|------|
| `=` | `=` | keycode変更 + Shift付加 |
| `'` | `'` | keycode変更 + Shift付加 |
| `` ` `` | `` ` `` | keycode変更 + Shift付加 |

### Shiftを維持するケース

US/JIS両側ともShiftあり（あるいは両側ともShiftなし）でkeycodeのみ変更:

| US入力 | 出力文字 | 備考 |
|--------|---------|------|
| `[` | `[` | Shiftなし→Shiftなし、keycode 0x2F→0x30 |
| `\` | `¥` | Shiftなし→Shiftなし、keycode 0x31→0x89 |
| `]` | `]` | Shiftなし→Shiftなし、keycode 0x30→0x32 |
| Shift+`7` (`&`) | `&` | Shiftあり→Shiftあり、keycode 0x24→0x23 |
| Shift+`8` (`*`) | `*` | Shiftあり→Shiftあり、keycode 0x25→0x34 |
| Shift+`9` (`(`) | `(` | Shiftあり→Shiftあり、keycode 0x26→0x25 |
| Shift+`0` (`)`) | `)` | Shiftあり→Shiftあり、keycode 0x27→0x26 |
| Shift+`=` (`+`) | `+` | Shiftあり→Shiftあり、keycode 0x2E→0x33 |
| Shift+`[` (`{`) | `{` | Shiftあり→Shiftあり、keycode 0x2F→0x30 |
| Shift+`\` (`\|`) | `\|` | Shiftあり→Shiftあり、keycode 0x31→0x89 |
| Shift+`]` (`}`) | `}` | Shiftあり→Shiftあり、keycode 0x30→0x32 |
| Shift+`'` (`"`) | `"` | Shiftあり→Shiftあり、keycode 0x34→0x1F |
| Shift+`` ` `` (`~`) | `~` | Shiftあり→Shiftあり、keycode 0x35→0x2E |
| Shift+`-` (`_`) | `_` | Shiftあり→Shiftあり、keycode 0x2D→0x87 |

### Altを除去するケース

| US入力 | 出力 | 操作 |
|--------|------|------|
| Alt+`` ` `` | 全角/半角 | Alt除去、keycode 0x35そのまま |

## 同時押し（6KRO）における注意事項

HIDレポートには最大6キーのkeycodeが含まれる。変換は各keycodeに対して
独立して適用するが、モディファイアは1つのバイトで全keycodeを共有する。

このため、Shiftの付加/除去が矛盾するキーを同時押しした場合（例:
`[` と `'` を同時押し → 一方は「Shiftなし」、他方は「Shift付加」が必要）
は正しく変換できない。これはUS→JIS変換アダプタの構造的な制限であり、
通常の入力では発生しないケースとして許容する。

参考実装: [m47ch4n/qmk-translate-ansi-to-jis](https://github.com/m47ch4n/qmk-translate-ansi-to-jis)
