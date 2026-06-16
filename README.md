# aerogu34

aerogu34 は Seeeduino XIAO BLE (nRF52840) ベースの 34 キー左右分割ワイヤレスキーボードです。
右側 (セントラル) に PAW3222 トラックボールを搭載し、JIS 配列を OS 設定で前提とした日本語キーマップを使用します。

## 主な特徴

- **34 キー**: 各サイドに 5×3 + 親指 2 キー
- **左右分割 + ワイヤレス**: BLE で接続。USB は右 (セントラル) のみ
- **トラックボール**: 右側に PAW3222
- **RGB LED ステータスウィジェット**: バッテリーレベル/接続状態を表示
- **DYA Studio 対応**: BLE 接続管理、バッテリー履歴、設定 RPC、トラックボール設定をブラウザから変更可能
  ([DYA Studio](https://studio.dya.cormoran.works/))
- **Prospector status advertisement 対応**:
  [Prospector Scanner](https://github.com/carrefinho/prospector-zmk-module) で
  キーボードの状態を別デバイスから観察可能

## ハードウェア

- マイコン: [Seeeduino XIAO BLE](https://wiki.seeedstudio.com/XIAO_BLE/) (nRF52840)
- トラックボール (右側のみ): PAW3222
- マトリクス: col2row、各 4 行 × 5 列

GPIO ピン割り当ては
[`boards/shields/aerogu34/aerogu34.dtsi`](boards/shields/aerogu34/aerogu34.dtsi)
を参照。

## キーマップ

[`config/aerogu34.keymap`](config/aerogu34.keymap) で定義。レイヤー構成:

| Layer | 用途 |
|---|---|
| 0 Default | 基本入力 |
| 1 Function | ファンクションキー、記号、矢印キー |
| 2 Number | 数字、四則演算、IME 切替 |
| 3 Scroll | BT プロファイル選択、トラックボールスクロール、bootloader/`studio_unlock` 起動 |
| 4 Mouse | マウスボタン |

OS 側の設定を**日本語 (JIS) キーボード**にした状態で正しい記号が打てる
ように、`JP_*` 系の defines で変換しています。

### 主なコンボ

| Keys | Result |
|---|---|
| `Q` + `W` (positions 11, 12) | `Tab` |
| `W` + `E` (positions 12, 13) | `Shift+Tab` |
| 親指左 2 + その隣 | `Left Win` |
| `0` + `4` キー (layer 3) | BT プロファイル全消去 |
| `1` + `2` + `3` キー (layer 3) | bootloader 起動 |

詳細は keymap ファイルを参照。

## ファームウェアのビルド

### GitHub Actions (推奨)

このリポジトリへ push すると
[.github/workflows/build.yml](.github/workflows/build.yml) が自動的に
[`build.yaml`](build.yaml) の全エントリをマトリクスビルドします。
GitHub Actions の Artifacts から `firmware` をダウンロードしてください。

ビルドターゲット:

| Artifact | 内容 |
|---|---|
| `aerogu34_right_prospector.uf2` | 右 (セントラル)、Prospector 有効 |
| `aerogu34_right.uf2` | 右 (セントラル)、Prospector 無効 |
| `aerogu34_left.uf2` | 左 (ペリフェラル) |
| `settings_reset.uf2` | BLE ペアリング情報リセット用 |

### ローカルビルド

[urob 氏の zmk-config workspace](https://github.com/urob/zmk-config)
ベースのビルド環境を想定しています。
Nix + west + Zephyr SDK が必要。

```bash
just init   # west workspace 初期化
just build aerogu34_right_prospector
just build aerogu34_right
just build aerogu34_left
just build settings_reset
```

または west 直接:

```bash
west build -s zmk/app -b seeeduino_xiao_ble \
  -S studio-rpc-usb-uart \
  -- -DSHIELD="aerogu34_right rgbled_adapter" \
     -DZMK_CONFIG=$(pwd)/config \
     -DZMK_EXTRA_MODULES=$(pwd)
```

## 書き込み (フラッシュ)

1. USB ケーブルでマイコンを PC に接続
2. リセットボタンを **2 回素早く** 押すと `XIAO-SENSE` または `XIAO-BLE` という
   USB ドライブとして認識される (DFU モード)
3. 対応する `.uf2` をドラッグ&ドロップ
4. 自動再起動して反映

新しい構成に切り替える際 (例: 別バリアントへ移行、GATT 構造が変わった等) は、
先に `settings_reset.uf2` を両側に書いて BLE ペアリング情報をクリアすると安全。

## 設定変更 (DYA Studio)

`aerogu34_right_prospector.uf2` または `aerogu34_right.uf2` どちらでも
DYA Studio 経由でランタイム設定が可能:

1. Chrome で [DYA Studio](https://studio.dya.cormoran.works/) を開く
2. USB を選択してセントラル (右側) を選ぶ
3. トラックボール感度、自動マウスレイヤー、idle/sleep タイムアウトなどを編集
4. 保存すると nRF52840 の settings 領域に persistent 保存される

BLE 接続は Prospector の advertising と競合するため、現状は USB 接続のみ
動作確認済み (詳細は後述の "既知の制限")。

## 依存モジュール / ライセンス

このプロジェクトは [MIT License](LICENSE) で配布されます (`Copyright (c) 2026 Tadashi Ogura`)。
ビルドに必要な外部依存とそのライセンスは以下の通りです。すべてビルド時に
[`config/west.yml`](config/west.yml) 経由で取得されるため本リポジトリ内では
再配布していませんが、謝辞および注意点として明示します。

| Module | License | Origin |
|---|---|---|
| [ZMK firmware (cormoran fork)](https://github.com/cormoran/zmk) | MIT | The ZMK Contributors |
| [prospector-zmk-module](https://github.com/carrefinho/prospector-zmk-module) | MIT | carrefinho 2024 + Prospector ZMK Module Contributors 2025 |
| [zmk-driver-paw3222](https://github.com/sekigon-gonnoc/zmk-driver-paw3222) | **Apache 2.0** | Google LLC 2024 + sekigon-gonnoc 2025 (modifications) |
| [zmk-rgbled-widget](https://github.com/caksoylar/zmk-rgbled-widget) | MIT | Cem Aksoylar 2023 |
| [zmk-module-ble-management](https://github.com/cormoran/zmk-module-ble-management) | MIT | cormoran 2026 |
| [zmk-module-battery-history](https://github.com/cormoran/zmk-module-battery-history) | MIT | cormoran 2026 |
| [zmk-module-settings-rpc](https://github.com/cormoran/zmk-module-settings-rpc) | MIT | cormoran 2026 |
| [zmk-module-runtime-input-processor](https://github.com/cormoran/zmk-module-runtime-input-processor) | MIT | cormoran 2026 |
| [zmk-input-processor-deadzone](https://github.com/t-ogura/zmk-input-processor-deadzone) | MIT | Tadashi Ogura 2026 |

### ライセンス上の注意点

- **paw3222 ドライバは Apache 2.0** であり、本リポジトリ (MIT) とライセンスが異なります。
  Apache 2.0 は MIT と互換 (MIT プロジェクトに取り込み可能) ですが、
  使用にあたって元の著作権表示と LICENSE 全文の保持が必要です。
  上記表での明示と、`west update` 時にモジュールに同梱される `LICENSE` ファイルが
  これに該当します。
- 他の依存はすべて MIT で、上記表での出自明示で十分です。
- Apache 2.0 / MIT 双方の本文と互換性については以下を参照:
  - <https://www.apache.org/licenses/LICENSE-2.0>
  - <https://opensource.org/license/mit>
  - <https://www.apache.org/licenses/GPL-compatibility.html>

## 既知の制限

### DYA Studio BLE 経由の接続が動作しない

Prospector 有効バリアント (`aerogu34_right_prospector.uf2`) で、
Web Bluetooth 経由の DYA Studio セッションが確立できません。
USB 接続なら問題ありません。

技術的な背景: Prospector v2.2.x は legacy advertising API を使用しているため、
ZMK Studio の directed advertising と単一の advertising slot を取り合います。
Prospector を完全無効化したい場合は `aerogu34_right.uf2` を使用してください。

### Prospector Scanner で見え方が不安定なケース

Prospector v2.2.1 には split central キーボードで Burst/Silent サイクルが
正しく終了しない不具合があり、その結果 ~10% しか advertising されません
([carrefinho/prospector-zmk-module #?](https://github.com/carrefinho/prospector-zmk-module)
 にて report 済)。
修正がマージされて `revision: main` に流れてくれば自動的に改善されます。
当面は scanner が間欠的に見える程度で機能上の問題はありません。

## 参考実装・参照リンク

- [ZMK firmware](https://zmk.dev) — 本体ドキュメント、Behavior 一覧、Build instructions
- [urob/zmk-config](https://github.com/urob/zmk-config) — Justfile + Nix によるビルド環境のベース
- [DYA Studio 統合ガイド](https://github.com/cormoran/zmk-keyboard-dya-dash) — cormoran 氏のリファレンス実装
- [DYA Studio](https://studio.dya.cormoran.works/) — ブラウザベースの ZMK Studio 拡張
- [Prospector ZMK Module](https://github.com/carrefinho/prospector-zmk-module) — Status advertisement
- 同種の Split + トラックボール + DYA Studio 構成の参考実装:
  - [zmk-config-KUKEY42](https://github.com/cormoran/zmk-config-KUKEY42)
  - [zmk-config-LiNEA40](https://github.com/cormoran/zmk-config-LiNEA40)

## ライセンス

[MIT License](LICENSE) — Copyright (c) 2026 Tadashi Ogura
