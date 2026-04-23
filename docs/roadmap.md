# picoremote 実装ロードマップ

実装の順序、依存関係、フェーズ分け、ブロッカー TODO をまとめる。
設計上の選択論は [`design-considerations.md`](./design-considerations.md) を参照。

## ゴール

PicoRuby (R2P2) を載せた Pico に対して、mpremote 並みの開発体験を提供する CLI を作る。

## 現状

- 調査・設計フェーズ完了
- 実装はこれから
- 言語 Ruby、gem 名 `picoremote`、対象 RP2040 / RP2350

## MVP (v0.1) — auto-connect + `.rb` 実行

README にも書いたとおり、最低限：

> 手元の `.rb` ファイルを実機に送って実行できる

ところまで。

### 依存関係グラフ

```
              ┌─ DeviceFinder ──┐
              │                 │
SerialPort ───┼─ SerialTransport ─┬─ PicoModemClient (FILE_WRITE) ──┐
              │                 │                                   │
              │                 └─ ShellDriver (shell↔irb, load) ───┤
              │                                                     │
              └─────────────────────────────────────────────────────┴─ Commands::Run
                                                                       Commands::Devs
                                                                       Commands::Connect
```

### v0.1 タスク（実装順）

1. **gem 雛形**（`Gemfile`, `picoremote.gemspec`, `bin/picoremote`, `lib/picoremote/version.rb`, `Rakefile`, `README` のセットアップ）
2. **`SerialPort` ラッパ**（`serialport` gem を内部で使う薄い抽象。open/close/read/write/timeout）
3. **`DeviceFinder`**（USB シリアル列挙 → R2P2 の VID/PID マッチ → 複数候補なら一覧表示）
   - **要先行: 実機の VID/PID 採取**（後述 TODO）
4. **`Commands::Devs`**（`picoremote devs` で 3 の出力を表示するだけ。実装コスト最小、デバッグに必要）
5. **`Commands::Connect`**（`auto` / `id:<sn>` / `port:<path>` / 直接パス。disconnect 含む）
6. **`PicoModemClient` の FILE_WRITE 実装**
   - フレーム encode/decode（STX + Length + Cmd + Payload + CRC16）
   - CRC16/CCITT, CRC32 の実装（or `digest-crc` gem）
   - チャンク送信 + CHUNK_ACK 待ち + DONE_ACK 検証
   - エラー / タイムアウト処理
   - **要先行: ループバックテスト or rp2040js での動作確認**
7. **`ShellDriver` 最小版**
   - シェルプロンプトを待つ
   - `irb\n` で IRB に切り替え、IRB プロンプトを待つ
   - 1 行コードを送る → 完了マーカー / プロンプトを待つ → stdout を吸い出す
   - `exit` で IRB を抜ける
   - **要先行: シェル / IRB のプロンプト仕様採取**
8. **`Commands::Run`**
   - 受け取った `<file.rb>` を `/tmp/picoremote-<hash>.rb` に FILE_WRITE
   - シェル → IRB に遷移
   - `load "/tmp/picoremote-<hash>.rb"\n` を送信
   - 実行中の stdout をホスト側に流す
   - 完了検知して IRB を抜ける
   - 後始末: 一時ファイルを削除（オプション、デフォルト ON）
9. **CLI ディスパッチ**（`OptionParser` ベースで最小限。`run` / `devs` / `connect` / `disconnect` / `version` / `help`）
10. **手動 E2E**: 実機で `picoremote run examples/hello.rb` が動く

### v0.1 完了の定義

```sh
$ picoremote devs
/dev/tty.usbmodem101  E6612483CB1BAF25  RP2040  R2P2

$ picoremote run examples/hello.rb
Detecting device... /dev/tty.usbmodem101
Transferring hello.rb (42 bytes)... done
Executing...
Hello, PicoRuby!
Done.
```

### v0.1 のスコープ外

- 対話シェル（`picoremote shell`）
- `cp` / `ls` / `cat` / `rm` / `mkdir`
- `eval` / `exec` の単体コマンド
- `reset` / `bootloader`
- CDC1 のタップ
- DFU
- 設定ファイル
- コマンドチェイン（`+` 区切り）

## v0.2 — 単発の FS 操作と対話 UX

11. **`PicoModemClient` の FILE_READ**（→ `cp` 両方向対応）
12. **`Commands::Cp`**（`:` プレフィックスでローカル/リモート判別、mpremote 互換）
13. **`Commands::Ls` / `Cat` / `Rm` / `Mkdir`**（IRB + 境界マーカー方式）
    - **要先行: PicoRuby に `JSON` があるか確認、無ければ inspect パーサを書く**
14. **`Commands::Reset`**（`Machine.reset` を IRB から実行）
    - **要先行: R2P2 の API 確認**
15. **`Commands::Shell`**（CDC0 の対話モニタ、Ctrl-] / Ctrl-x で抜ける、Ctrl-B 衝突対策）

## v0.3 — 開発体験の上乗せ

16. **`Commands::Eval` / `Commands::Exec`**（IRB 行送り + 結果取得）
17. **`Commands::Edit`**（cat → `$EDITOR` → 書き戻し）
18. **CDC1 ログ**（`picoremote log --cdc1`、`picoremote shell --tap-cdc1`）
19. **ユーザー設定ファイル**（`~/.config/picoremote/config.rb`、ショートカット展開）
20. **コマンドチェイン**（`+` 区切り、暗黙の `connect auto`）

## v0.4+ — 任意

21. **`Commands::Dfu`**（PicoModem の DFU_START 経由でファーム更新）
22. **`mount` 相当**（R2P2 側に RemoteFS ドライバを注入する。大工事）
23. **パッケージ管理**（PicoRuby 向け mip 相当が整備された後）

## MVP ブロッカー TODO

v0.1 着手の前に解決したい順:

- [ ] **実機の USB VID/PID を採取**（DeviceFinder の判定条件確定）
  - macOS: `system_profiler SPUSBDataType | grep -A 15 -i pico`
  - Linux: `lsusb -v` / `udevadm info -a -n /dev/ttyACM0`
  - dual CDC のうちどれがメイン (CDC0) かも採取
- [ ] **シェル / IRB のプロンプト文字列を採取**（ShellDriver の待ち条件確定）
  - シェル: 起動直後 / カレントディレクトリ変更後 / 直前コマンドの戻り値での見え方
  - IRB: 通常プロンプト / 複数行継続時のプロンプト
- [ ] **`picoruby-picomodem` のフレーム実装と完全互換になるかループバックで検証**
  - 試験用に `test/fixtures/` に R2P2 側の応答ログを置けるようにする
- [ ] **gem 名の重複確認**
  - rubygems.org で `picoremote` / `pico-remote` / `pico_remote` を検索
- [ ] **`serialport` gem の挙動確認**
  - macOS / Linux でのタイムアウト挙動
  - 非ブロッキング read 周り
  - メンテ状況（必要なら `rubyserial` も比較）

## v0.2 以降で必要になる調査 TODO

- [ ] R2P2 側に `Machine.reset` / `Machine.bootloader` 相当 API があるか確認
- [ ] PicoRuby に `json` gem が入っているか（`Commands::Ls` のパース方式に影響）
- [ ] `Ctrl-B` をホスト側のエスケープと衝突させない UX のデフォルト挙動

## いつでも良い TODO

- [ ] 統合テスト基盤（シリアルループバック / rp2040js / モック）
- [ ] CI 設計（GitHub Actions、macOS / Linux マトリクス）
- [ ] リリース手順（`rake release` で rubygems.org に push）
