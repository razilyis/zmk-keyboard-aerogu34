# 作業履歴 (History)

## 2026-07-08
### リポジトリの把握
- **プロジェクト概要**: Seeed XIAO BLE (nRF52840) をベースにしたカスタム分割キーボード「Aerogu34」の ZMK ファームウェア構成。
- **特徴的なモジュール**:
  - `cormoran` 氏のフォーク版 ZMK (`main+dya` ブランチ) をベースに、Web GUI (DYA Studio) との連携を実装。
  - PIXART PAW3222 トラックボールドライバ (`sekigon-gonnoc/zmk-driver-paw3222`) を統合し、右側 (Central) に接続。
  - `prospector-zmk-module` を用いて、BLE ステータス・WPM などのアドバタイズ・テレメトリ情報を送信可能。
  - `zmk-rgbled-widget` を使用して LED によるバッテリー残量などのステータス表示が可能。
  - `zmk-input-processor-deadzone` によるトラックボールの微細なノイズ除去。
  - トラックボール操作時に自動でマウスレイヤー (レイヤー 4) に遷移する `zip_temp_layer` や、スクロールレイヤー (レイヤー 3) でのスクロール変換 `scroll_runtime_input_processor` を導入。
- **ビルド構成 (`build.yaml`)**:
  - `aerogu34_right_prospector.uf2`: 右側 (Central)、Prospector 有効
  - `aerogu34_right.uf2`: 右側 (Central)、Prospector 無効 (Web Bluetooth での DYA Studio 接続用)
  - `aerogu34_left.uf2`: 左側 (Peripheral)
  - `settings_reset.uf2`: BLE ペアリング情報リセット用

### DYA StudioのBluetooth接続フリーズ問題への対策
- **問題事象**: DYA StudioにBluetooth接続してキーマップ変更を行おうとするとフリーズする。
- **原因の分析**:
  - DYA Studio用のカスタムRPCモジュール（バッテリー履歴、BLE管理、インプットプロセッサ設定など）が追加されているため、ZMKのデフォルト設定のままだとZMK StudioのRPCスレッドのスタック領域や送受信バッファが不足し、キーマップ等の大容量データ送信時にスタックオーバーフローやバッファ詰まりが発生してフリーズしている可能性が高い。
  - トラックボール処理等のカスタムインプットプロセッサが動作する際に入力スレッドのスタック容量（デフォルト512〜1024バイト）を圧迫している可能性、およびBLE書き込み時の電波強度不足も影響する。
- **実施した対策**:
  - [config/aerogu34_right.conf](file:///E:/OneDrive%20-%20%E3%82%B9%E3%82%BF%E3%83%83%E3%83%95%E3%83%9E%E3%83%BC%E3%82%B1%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0%E6%A0%AA%E5%BC%8F%E4%BC%9A%E7%A4%BE/github/zmk-keyboard-aerogu34/config/aerogu34_right.conf) を修正し、以下のバッファ、スレッドスタックの最適化設定を追記。
    - `CONFIG_ZMK_STUDIO_RPC_THREAD_STACK_SIZE=4096`（ZMK Studio RPCスレッドのスタックを4096バイトへ拡張）
    - `CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=256`（受信バッファを256バイトへ拡張）
    - `CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256`（送信バッファを256バイトへ拡張）
    - `CONFIG_INPUT_THREAD_STACK_SIZE=2048`（入力処理スレッドのスタックを2048バイトへ拡張）
    - `CONFIG_ZMK_SPLIT_BLE_CENTRAL_SPLIT_RUN_STACK_SIZE=2048`（分割中央書き込みスレッドのスタックを2048バイトへ拡張）
    ※ `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` は消費電力への配慮等の観点から除外。


