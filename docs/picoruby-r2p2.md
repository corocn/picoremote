# PicoRuby / R2P2 調査メモ

picoremote が制御対象とする PicoRuby 環境（R2P2 シェル）側の仕様調査。mpremote と同じ設計をそのまま当てはめられない箇所が多いので、差分の把握が目的。

- リポジトリ（統合後）: <https://github.com/picoruby/picoruby>
- R2P2 gem: `mrbgems/picoruby-r2p2` / `mrbgems/picoruby-shell` / `mrbgems/picoruby-picomodem`
- ターミナル設定: <https://picoruby.org/terminal-emulator>

## 1. R2P2 とは

R2P2 = Raspberry Pi Pico 上で動く **PicoRuby 製の Unix ライクシェル**。MicroPython の「REPL がそのまま本体」という発想ではなく、**シェルと IRB が別モード**になっているのが最大の違い。

- `/` 配下に LittleFS (`:flash`) が mount される
- `/bin/` に Ruby で書かれたコマンド（`ls`, `cp`, `rm`, `mv`, `cat`, `mkdir`, `touch`, `df`, `vim`, `irb`, `rapicco`, `dfucli`, `dfuble`, ... ）
- `/etc/init.d/r2p2` で起動スクリプト、`/etc/config.yml` で ENV/デバイス設定
- `/home/app.rb` があると自動実行（autostart）
- `$PATH`, `$HOME` などの環境変数が通る

## 2. 対応ボードとビルド

| Board | Chip | WiFi | femtoruby (mruby/c) | picoruby (mruby) |
|---|---|---|---|---|
| pico | RP2040 | × | ✓ | - |
| pico_w | RP2040 | ✓ | ✓ | - |
| pico2 | RP2350 | × | ✓ | ✓ |
| pico2_w | RP2350 | ✓ | ✓ | ✓ |

ビルドは `rake r2p2:{vm}:{board}:{mode}` の形式。UF2 で配布される。

## 3. USB 接続仕様

シリアル設定:

- ボーレート: 115200
- データ 8 / パリティなし / ストップ 1 / フロー制御なし
- New-line: 送受信とも LF 推奨
- macOS では screen 系が不安定、GTKTerm(Linux)/TeraTerm(Windows) 推奨

**重要**: R2P2 は **Dual CDC** 構成。

- `CDC 0`（Linux 例: `/dev/ttyACM0`）: メインターミナル（シェル対話 + アプリの stdout）
- `CDC 1`（`/dev/ttyACM1`）: デバッグ出力（stderr、`Machine.debug_puts`）— **debug ビルドのみ**

picoremote からは基本 CDC0 に接続すれば十分だが、デバッグ用途で両方開ける選択肢は用意したい。

## 4. Web Terminal

R2P2 3.4 以降は Web Serial API 経由で <https://picoruby.org/terminal> がブラウザからそのまま使える（同じプロトコルを picoremote 側で喋る参考になる）。

## 5. シェルの入力ハンドリング（`Shell#run_shell`）

抜粋（`mrbgems/picoruby-shell/mrblib/shell.rb`）:

- **Ctrl-B (0x02, STX)**: **PicoModem（RBTP）セッションへ遷移**。ホストに `\x06`(ACK) を返してから `PicoModem.session($stdin, $stdout)` を呼ぶ。
- **Ctrl-C**: 入力バッファクリア
- **Ctrl-D**: バッファが空ならメッセージのみ、非空なら Enter 相当（コマンド実行）
- **Ctrl-Z**: `Machine.exit(0)`（※シェル終了）
- LF/CR: コマンドラインをパースして実行

IRB モード（`irb` コマンドで起動）の入力:

- **Ctrl-C**: バッファクリア
- **Ctrl-D**: 空ならログアウト
- **Ctrl-Z**: サスペンド（`Signal.raise(:TSTP)`）
- LF/CR: サンドボックスで compile → 成功すれば exec、失敗（シンタックス不完全）なら継続入力

**注意**: MicroPython のような「Ctrl-A で Raw モード」は存在しない。ホストから安定してコードを流し込みたければ **PicoModem か、IRB に `load "/tmp/xxx.rb"` を叩かせる** のが現実的。

## 6. PicoModem / R2P2 Binary Transfer Protocol (RBTP)

`mrbgems/picoruby-picomodem/mrblib/picomodem.rb` に全実装。mpremote の「mount 用 FS hook プロトコル」に近いが、より汎用的で、フレーム境界が明確。

### 6.1 トリガ

- ホスト → デバイス: `\x02`（STX / Ctrl-B）をシェルに 1 バイト送る
- デバイス → ホスト: `\x06`（ACK）を返し、`STDIN.raw!` の上で `PicoModem.session` が動き始める

### 6.2 フレーム形式

```
+------+--------+-----+---------+-------+
| STX  | Length | Cmd | Payload | CRC16 |
| 0x02 |  2 B   | 1 B |   N B   |  2 B  |
|      |  BE    |     |         |  BE   |
+------+--------+-----+---------+-------+

Length = sizeof(Cmd) + sizeof(Payload) = 1 + N
CRC16  = CRC-16/CCITT over (Cmd || Payload)
```

### 6.3 コマンド

| 方向 | 名前 | 値 | 意味 |
|---|---|---|---|
| H→D | `FILE_READ` | 0x01 | デバイス上のファイルをホストに送る |
| H→D | `FILE_WRITE` | 0x02 | ホストからのファイルをデバイスに書く |
| H→D | `DFU_START` | 0x03 | DFU（ファーム更新）開始 |
| H→D | `CHUNK` | 0x04 | チャンク転送（WRITE/DFU のペイロード） |
| H→D | `DONE` | 0x0F | 完了通知 |
| H→D | `ABORT` | 0xFF | 中断 |
| D→H | `FILE_DATA` | 0x81 | ファイル読み出しチャンク |
| D→H | `FILE_ACK` | 0x82 | FILE_WRITE の受付応答 |
| D→H | `DFU_ACK` | 0x83 | DFU_START の受付応答 |
| D→H | `CHUNK_ACK` | 0x84 | 各 CHUNK の ACK |
| D→H | `DONE_ACK` | 0x8F | 完了応答（最後に CRC32 を載せる） |
| D→H | `ERROR` | 0xFE | エラー |

ステータスコード: `OK = 0x00`, `READY = 0x01`, `FAIL = 0xFF`。

デフォルトパラメータ:

- チャンクサイズ: **480 バイト**
- タイムアウト: 5000 ms（`read_exact` で 1ms ごとに waited カウント）

### 6.4 FILE_READ のシーケンス

```
H: STX | len | FILE_READ(0x01) | path                   | crc
D: STX | len | FILE_DATA(0x81) | total_size(4,BE) | chunk1 | crc
H: STX | len | CHUNK_ACK(0x84) | OK                     | crc
D: STX | len | FILE_DATA(0x81) | chunk2                 | crc
H: STX | len | CHUNK_ACK(0x84) | OK                     | crc
  ... 最後まで繰り返し ...
D: STX | len | DONE_ACK(0x8F)  | OK | file_crc32(4,BE)  | crc
```

- 最初の FILE_DATA にだけ `total_size`（4B BE）が先頭に乗る。
- 送信側（デバイス）は CRC32 をチャンクごとにインクリメンタル計算して最後に渡す。
- ホストは全バイトを受け取ったら自分でも CRC32 を計算して検証できる。

### 6.5 FILE_WRITE のシーケンス

```
H: STX | len | FILE_WRITE(0x02) | total_size(4,BE) | path | crc
D: STX | len | FILE_ACK(0x82)   | READY                  | crc
H: STX | len | CHUNK(0x04)      | chunk1                 | crc
D: STX | len | CHUNK_ACK(0x84)  | OK                     | crc
  ... total_size まで繰り返し ...
D: STX | len | DONE_ACK(0x8F)   | OK | file_crc32(4,BE)  | crc
```

デバイス側は受信後に `File.open(path, "w")` で書き込み、可能なら `f.expand(size)`（LittleFS の事前確保）を呼ぶ。

### 6.6 DFU_START

- ペイロードは「DFU ヘッダ（19B + 任意の署名）」
- デバイスは `DFU::Updater.expected_size(header)` でボディサイズを決定し、`READY` を返したあと CHUNK で受信
- 受信完了後、`DFU::Updater#receive` にメモリバッファとして渡して書き込ませる
- その間 `$stdout` は NullIO に差し替えられる（フレームを壊さないため）

### 6.7 セッション終了

- 1 つのコマンドを処理し終えると `break` してセッション脱出
- `STDIN.cooked!` に戻し、200ms 待ってから `[PicoModem] <info>` を stdout に print → シェルに復帰

picoremote が PicoModem を使う場合、**1 セッション = 1 操作** のサイクルを繰り返すことになる（mpremote の raw REPL のような「連続実行の常駐モード」ではない）。

## 7. シェルから使えるビルトインコマンド（抜粋）

`picoruby-shell/shell_executables/` に並ぶ Ruby スクリプト。これらは FS 上の `/bin/*.rb` として展開され、`$PATH` 経由で呼ばれる。

| コマンド | 用途 |
|---|---|
| `ls`, `cd`, `pwd`, `cat`, `cp`, `mv`, `rm`, `mkdir`, `touch`, `head`, `tail`, `df`, `free`, `date`, `uptime`, `taskstat` | 基本 Unix ライクコマンド |
| `irb` | 対話 Ruby |
| `vim` | 簡易テキストエディタ |
| `install` | `/bin` にスクリプトをインストール |
| `ntpdate`, `setup_rtc` | 時刻同期 / RTC セットアップ |
| `wifi_connect`, `wifi_disconnect`, `ifconfig`, `nmcli`, `nmble`, `ping` | ネットワーク |
| `dfucli`, `dfucat`, `dfuble`, `dfutcp` | DFU アップデート各種トランスポート |
| `hello`, `rapicco`, `r2p2` | デモ/管理 |
| `setup_sdcard` | SD カード初期化 |

picoremote 側から `ls` / `cat` を叩いてその **stdout をテキストパース** するという方式も選べる（PicoModem を使いたくない場面向け）。ただし端末制御コード（ANSI カラー・カーソル）が混ざる前提があるので、堅牢性では PicoModem に劣る。

## 8. デバイス判別

- USB VID/PID は公式情報を要確認（リリースバイナリのベンダ設定次第）
- `Machine.unique_id` でデバイス固有 ID 取得可能 → ホスト側では `pyserial` の `serial_number` で区別する（mpremote の `id:` と同じ手）
- dual CDC なので `id:<sn>` 指定に加えて **どちらの CDC か** を決める必要がある（VID/PID は共通・interface number で区別）

## 9. mpremote と比較した制約

| 項目 | mpremote (MicroPython) | picoremote (R2P2) で直面する現実 |
|---|---|---|
| 実行コード流し込み | Raw REPL / Raw Paste でフレーム化済み | Raw モード不在。IRB の行単位評価 or PicoModem で送って `load` |
| 結果取得 | `print(repr(x))` → `ast.literal_eval` | Ruby の `.inspect` は Python `repr` と互換でない値あり。JSON / YAML / Marshal も前提にできない場面あり |
| FS 操作 | Raw REPL 経由の exec/eval | PicoModem の FILE_READ/WRITE を優先。メタデータ(stat)は IRB 経由 or 専用コマンドが必要 |
| mount | デバイス側 VFS へ動的に RemoteFS 注入 | 同等機構は未実装。将来ドライバを PicoRuby 側に書けば再現可能 |
| soft-reset | Raw REPL 内で `\x04` | シェル再起動相当が必要。`Machine.reset` を IRB で叩く or ショートカット用意 |
| ターミナル抜け出し | `Ctrl-]`/`Ctrl-x` | 同じホスト側制御で実装する。Ctrl-B はデバイス側で予約（PicoModem）なので衝突注意 |
| 非印字バイト扱い | 生シリアル | CDC0 は ANSI エスケープ必須（シェル UI がカラー/カーソル制御を流す） |
| dual-port | 単一 | メイン/デバッグ を同時に扱える仕組みがあると便利 |

## 10. picoremote として拾いたいユースケース

1. **ポート列挙 / 自動接続**（`id:<sn>` / `port:<path>` / `auto`）
2. **ターミナル（シェル / IRB）モニタ**（Ctrl-B は PicoModem 予約なのでエスケープ必要）
3. **ファイル転送 (PicoModem FILE_READ / FILE_WRITE)** — `cp`, `ls`（IRB にワンショットで `Dir.entries` を叩かせる or 専用コマンド）
4. **リモート実行 (`run`)** — PicoModem で `/tmp/foo.rb` に置いてから IRB で `load`、または `send_lines` でラインずつ IRB 注入
5. **DFU デプロイ** — PicoModem の `DFU_START` を使う（`dfucli` 相当の置き換え）
6. **デバッグ CDC1 の並行タップ** — 副ポートを同時にオープンしてプレフィックス付きでまとめて出力
7. **`reset` / `bootloader` ショートカット** — IRB 経由で `Machine.reset` / 同等 API を呼ぶ
8. **config.py 互換のユーザーショートカット** — mpremote と同じ感覚で `commands = { "a0": "connect /dev/ttyACM0", ... }` を書ける

これらは [`design-considerations.md`](./design-considerations.md) にまとめる。
