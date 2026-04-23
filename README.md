# picoremote

PicoRuby (R2P2) を載せた Raspberry Pi Pico / Pico 2 シリーズへのデプロイ・実行・REPL 操作を楽にするための CLI ツール。
mpremote にインスパイアされた、PicoRuby 向けのリモートコントロールユーティリティ。

## Status

調査・設計フェーズ。実装はこれから。

調査メモは [`docs/`](./docs/) に置いてある:

- [`docs/mpremote.md`](./docs/mpremote.md) — mpremote の CLI / プロトコル (Raw REPL, Raw Paste, FS Hook) 調査
- [`docs/picoruby-r2p2.md`](./docs/picoruby-r2p2.md) — R2P2 / PicoModem (RBTP) の仕様
- [`docs/design-considerations.md`](./docs/design-considerations.md) — 設計方針のたたき台

## Target Hardware

R2P2 がサポートしているボード:

| Board | Chip |
|---|---|
| Raspberry Pi Pico | RP2040 |
| Raspberry Pi Pico W | RP2040 |
| Raspberry Pi Pico 2 | RP2350 |
| Raspberry Pi Pico 2 W | RP2350 |

動作確認対象は **RP2040 / RP2350** の両方。

## Host

macOS / Linux / Windows を想定（まずは macOS / Linux を優先）。

## Implementation

- 言語: **Ruby (CRuby gem)**
- 配布: `gem install picoremote` 想定
- 依存: シリアルポート操作 (`serialport` gem 等を検討)
- PicoRuby / R2P2 コミュニティと同じ言語にそろえることで、`picoruby-picomodem` と RBTP プロトコル定義を共有しやすくする狙い

## MVP スコープ (v0.1)

最低限、手元の `.rb` ファイルを実機に送り込んで実行できるところまで。

### 機能

1. **自動ポート検出 + 自動接続**
   - 接続されている USB シリアルから R2P2 が載った Pico を判別
   - VID/PID で自動絞り込み。複数ある場合は一覧提示
   - Dual CDC のうちメイン側 (CDC0) に繋ぐ
2. **`.rb` ファイルのデプロイ & 実行**
   - 指定したローカルの `.rb` ファイルを PicoModem (RBTP) で実機に転送
   - 実行完了まで stdout をターミナルに流す
   - 終わったら元のシェル状態に戻す

### CLI イメージ

```
$ picoremote run path/to/main.rb
Detecting device... /dev/tty.usbmodem101 (RP2040, R2P2)
Transferring main.rb (1234 bytes)... done
Executing...
Hello, PicoRuby!
Done.
```

### MVP で作るモジュール想定

```
lib/picoremote/
├── serial_port.rb       # シリアル抽象
├── device_finder.rb     # USB ポート列挙 + VID/PID マッチ
├── pico_modem.rb        # RBTP フレームエンコード/デコード (CRC16, CRC32)
├── shell_driver.rb      # シェル <-> IRB のモード遷移
└── commands/
    └── run.rb           # 転送 + IRB load の一連
exe/picoremote           # CLI エントリ
```

### MVP の対象外（将来フェーズ）

- インタラクティブな REPL/シェルモニタ
- `ls` / `cat` / `cp` / `rm` などの FS 操作
- `edit` / `eval` / `exec`
- DFU アップデート
- CDC1 (デバッグポート) の tap
- mpremote 互換のコマンドチェイン (`+` 区切り)
- ユーザー設定ファイル (`~/.config/picoremote/config.rb`)
- `mount` 相当の疑似 VFS

詳細な設計案とフェーズ分けは [`docs/design-considerations.md`](./docs/design-considerations.md) を参照。

## License

MIT (予定)
