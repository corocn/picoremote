# mpremote 調査メモ

MicroPython 公式の CLI ツール `mpremote` の調査結果。picoremote を設計する上でもっとも参考になる既存実装。

- 公式ドキュメント: <https://docs.micropython.org/en/latest/reference/mpremote.html>
- ソース: <https://github.com/micropython/micropython/tree/master/tools/mpremote>
- 言語: Python (`pip install mpremote` で配布)
- 依存: `pyserial`, `platformdirs`

## 1. 位置付けとコンセプト

`mpremote` は「シリアル接続された MicroPython デバイスに対して、対話/デプロイ/自動化を行う統合ユーティリティ」。

特徴:

- **コマンドチェイン方式**: 1 回の起動で複数コマンドを順番に実行できる（例: `mpremote connect a0 cp main.py : + reset + repl`）。
- **暗黙の `connect auto` と `repl`**: 引数なしで叩くと自動接続して REPL に入る。
- **暗黙の soft-reset**: デバイスを既知の状態に戻すため、最初の action 系コマンド実行前に 1 度だけ soft-reset する。`resume` で抑止可能。
- **exclusive モードでシリアルをオープン**: 多重起動は自動で別ポートへフォールバック。
- **CLI とライブラリ両用**: `import mpremote` でも使えることを目指している（APIは発展中）。

## 2. サブコマンド一覧

| コマンド | 役割 |
|---|---|
| `connect <device>` | `list` / `auto` / `id:<sn>` / `port:<path>` / `rfc2217://` / 任意のパスに接続 |
| `disconnect` | 切断（以降 auto soft-reset を再有効化） |
| `resume` | 次コマンドでの auto soft-reset を抑止 |
| `soft-reset` | Ctrl-A → Ctrl-D 相当の soft reboot |
| `repl` | 対話ターミナル（シリアルモニタ）。`--capture`, `--inject-code`, `--inject-file` |
| `eval <expr>` | Python 式を評価して結果を print |
| `exec <code>` | Python コードを実行（`--no-follow` で fire-and-forget） |
| `run <file>` | ローカルの .py を **RAM 上で** 実行（デバイスFSに置かない） |
| `fs <subcmd>` | `cat/ls/cp/rm/mkdir/rmdir/touch/sha256sum/tree` |
| `df` | ファイルシステム使用量（実態は `exec` のエイリアス） |
| `edit <file>` | リモートファイルを `$EDITOR` で編集して書き戻す |
| `mip install` | micropython-lib / GitHub / GitLab からパッケージを入れる |
| `mount <dir>` | ローカル dir を `/remote` としてマウント（擬似 VFS） |
| `umount` | アンマウント |
| `romfs query/build/deploy` | ROMFS パーティションの操作 |
| `rtc [--set]` | デバイス RTC の取得/設定 |
| `sleep <s>` | コマンド列中のウェイト |
| `reset` | hard reset（`machine.reset()` を exec するエイリアス） |
| `bootloader` | `machine.bootloader()` を呼んで UF2/DFU モードへ |
| `help` / `version` | ヘルプとバージョン |

### ショートカット（ショートカット定義は拡張可能）

- `a0`/`a1`/`a2`/`a3` → `connect /dev/ttyACM{n}`
- `u0..u3` → `/dev/ttyUSB{n}`
- `c0..c3` → `COM{n}`（Windows）
- `devs` → `connect list`
- `cat`/`cp`/`ls`/`rm`/`mkdir`/`rmdir`/`touch`/`sha256sum`/`tree`/`edit` → `fs <cmd>` のエイリアス
- ユーザー設定 `config.py`（`$XDG_CONFIG_HOME/mpremote/config.py` など）で任意の `commands = {...}` を定義可能
- 引数つきエイリアスも可（例: `"multiply x=4 y=7": "eval x*y"`）

### コマンド区切り `+`

- `fs cp` のようにファイル複数を取るコマンドの後に、次のコマンドを置きたい場合は `+` で区切る。

## 3. ソースコード構成

```
tools/mpremote/mpremote/
├── __init__.py          # バージョン
├── __main__.py          # エントリ
├── main.py              # 引数パーサ/コマンド駆動ループ/ユーザ設定ロード
├── commands.py          # do_connect/do_exec/do_filesystem/do_mount/... の実装
├── transport.py         # Transport 抽象クラスと fs_* 系の共通実装
├── transport_serial.py  # SerialTransport + 仮想FS hook (fs_hook_code)
├── console.py           # termios / Windows の端末モード制御
├── repl.py              # インタラクティブ REPL（Ctrl-] で抜ける）
├── mip.py               # micropython-lib / GitHub 等からのパッケージ取得
├── mp_errno.py          # MicroPython の errno → ホスト errno
└── romfs.py             # ROMFS イメージ作成/デプロイ
```

### State（`main.State`）

- `transport`: 現在の `SerialTransport`（未接続なら None）
- `_did_action`: action 系コマンドを実行したか（なければ最後に暗黙の `repl`）
- `_auto_soft_reset`: 次の raw REPL 侵入時に soft-reset するか

`ensure_connected` / `ensure_raw_repl` / `ensure_friendly_repl` で状態遷移を保証する。

## 4. MicroPython の Raw REPL プロトコル

mpremote は「print されたテキストをパースする」のではなく、**Raw REPL** という MicroPython 特有のプロトコルを使ってホストから確実にコード実行と結果取得を行う。

### Raw REPL モード遷移

| 操作 | バイト |
|---|---|
| 実行中プログラムの割り込み | `\r\x03`（Ctrl-C） |
| Friendly REPL → Raw REPL | `\r\x01`（Ctrl-A） |
| Raw REPL → Friendly REPL | `\r\x02`（Ctrl-B） |
| Soft reset (Raw REPL 内で) | `\x04`（Ctrl-D） |

- Raw REPL 進入時のバナー: `raw REPL; CTRL-B to exit\r\n>`
- soft reset 時のバナー: `soft reboot\r\n`

### 一発コード送信の流れ (`exec_raw_no_follow`)

1. プロンプト `>` を待つ。
2. `\x05A\x01` を送って **Raw Paste モード** を試みる。
   - レスポンス `R\x01` なら raw-paste でコードを書き込む。
   - `R\x00` なら raw-paste 非対応なので通常 Raw REPL にフォールバック。
3. 通常モードでは **256 バイト × 10ms** のレートでコードを流し、最後に `\x04` で完了。
4. `OK` を受信したら exec 受付完了。

### Raw Paste モード

デバイスが受け取り可能なウィンドウサイズ制御付きの書き込み。`struct "<H"` で window size を受け取り、`\x01` で window 増分、`\x04` で完了。高速でフロー制御ができる。

### 実行結果の読み取り (`follow`)

実行後デバイスは `stdout → \x04 → stderr → \x04` の順に返す。2 つの `\x04` で区切られた 2 ブロックを受けて、後者が非空なら `TransportExecError` を送出する。

### `eval`

ホスト側は `print(repr(<expr>))` を exec し、得られた文字列を `ast.literal_eval` して Python 値に戻している。つまり eval/exec は **常にデバイス上で `print` させてホストがパースする** モデル。

## 5. ファイルシステム操作

`transport.py` の `Transport` クラスに fs_* メソッド群がある。実体は Raw REPL 経由で exec/eval するラッパ。

### 代表例

- `fs_listdir(src)`: `for f in os.ilistdir(src): print(repr(f), end=',')` を exec し、ホスト側で `ast.literal_eval`。
- `fs_readfile`: `f=open(path,'rb'); r=f.read` してから `r(256)` を eval でチャンク取得。
- `fs_writefile`: `f=open(path,'wb'); w=f.write` してチャンクごとに `w(<bytes_repr>)` を exec。
- `fs_stat`: `os.stat` を eval で取得して `os.stat_result` に戻す。
- `fs_hashfile`: デバイス上に `hashlib.sha256` があれば計算させ、なければローカルで計算。
- `cp` は事前に SHA256 比較してスキップ（`--force` で強制）。

エラー時は `OSError` の文字列から errno 名をマッチさせて、ホスト側で `OSError(errno)` に復元している。**「ホストが Python コード片を生成 → デバイスで exec → print で結果をシリアル越しに返させる」** というのが統一された実装パターン。

## 6. `mount` の擬似 VFS（FS Hook）

`mpremote mount <local-dir>` は強力で特徴的な機能。

- デバイス側に `RemoteFS` という VFS ドライバと `RemoteCommand` クラスを含む Python コードを **Raw REPL 経由で注入**（ソース: `fs_hook_code` in `transport_serial.py`）。
- `os.mount(RemoteFS(...), "/remote")` + `os.chdir("/remote")` を実行。
- デバイス側は `/remote` 配下の open/read/write などを **専用のバイナリコマンド（CMD_STAT, CMD_OPEN, …）** としてシリアル出力し、ホスト側（`PyboardCommand`）がローカル FS を叩いて応答を返す。
- 通常のシリアル I/O と混ざらないように、リクエストは **`0x18` をシンクバイト** として使い、`SerialIntercept` が REPL 出力から切り分ける。
- コマンドコード:
  ```
  CMD_STAT=1, CMD_ILISTDIR_START=2, CMD_ILISTDIR_NEXT=3,
  CMD_OPEN=4, CMD_CLOSE=5, CMD_READ=6, CMD_READLINE=7,
  CMD_WRITE=8, CMD_SEEK=9, CMD_REMOVE=10, CMD_RENAME=11,
  CMD_MKDIR=12, CMD_RMDIR=13
  ```
- `fs_hook_code` は送信前にコメント除去・インデント圧縮・識別子短縮などで軽量化される。

仕組み的には **「デバイス上の VFS で RemoteFS が mount される → そのファイル操作がシリアルの専用フレームで ホストに向けて発行される → ホストがローカルディスクで処理して応答」** という双方向プロトコル。開発中に『コピーせずにホスト上のコードをそのまま走らせる』用途に便利。

## 7. REPL モード

`repl.py` は「素の」シリアルモニタ。

- `Ctrl-]` または `Ctrl-x` で抜ける
- `Ctrl-D`: soft-reset（mount 中は再マウントを自動実施）
- `Ctrl-J`: `--inject-code` で指定した文字列を送信
- `Ctrl-K`: `--inject-file` のファイルを Raw REPL 経由で一発実行
- `--capture <file>` で受信をファイルに保存
- `--escape-non-printable` で非印字バイトを `[xx]` 表示

Ctrl-D 処理の際、mount 状態を検知して soft reboot バナー（`MPY: soft reboot` と `MicroPython ` バナー）を待ち、再度 `fs_hook_code` を注入し直すロジックがある。

## 8. 接続・デバイス選択

`do_connect` は以下をサポート:

- `list`: `pyserial.tools.list_ports` を列挙して `device sn vid:pid manufacturer product` を表示
- `auto`: VID/PID を持つ最初のシリアルへ接続
- `id:<sn>`: USB シリアル番号で選択
- `port:<path>` または素のパス: 直接指定
- `rfc2217://host:port`: ネットワーク越しのシリアル

baudrate は固定で 115200。Windows の CP210x ドライバ向けのリセット回避処理あり（Silicon Labs VID の場合 DTR/RTS の順序を制御）。

## 9. ユーザーコンフィグ

- 位置: `platformdirs.user_config_dir("mpremote", appauthor=False)/config.py`
- 形式: Python ファイルで `commands = {...}` を定義
- 例: `"c33": "connect id:334D335C3138"`, `"test": ["mount", ".", "exec", "import test"]`
- 引数パターン: `"double x=4": "eval x*2"` で `mpremote double 10` のように呼べる

## 10. `mip` パッケージ管理

- デフォルトで micropython-lib の index から取得
- `github:org/repo[@ref]`, `gitlab:org/repo[@ref]` 記法をサポート
- `--target`, `--index`, `--mpy` オプション
- 可能ならデバイス上で `.mpy` を使う

## 11. picoremote 目線での要点

| mpremote の機構 | PicoRuby で置き換え検討する対応物 |
|---|---|
| Raw REPL (`Ctrl-A`, `OK`, `\x04` 区切り) | R2P2 シェルの IRB モードをスクレイピングするか、独自プロトコル(PicoModem)を使うか |
| Raw Paste モード（フロー制御） | RBTP の CHUNK/CHUNK_ACK が相当 |
| `fs_*` を全部 `print(repr(...))` で取る | R2P2 では shell コマンド (`ls`, `cat`) のテキスト出力 or PicoModem の FILE_READ/FILE_WRITE |
| `mount` の FS Hook | PicoRuby 側に同等ドライバを書けば再現可能だが、まず PicoModem ベースの `cp` を優先検討 |
| `run <file>` (RAM 実行) | IRB へ `load "path"` 相当か、PicoModem 経由で一時ファイル置いて実行か検討 |
| `connect auto` | 同じく `pyserial.tools.list_ports` で USB VID を見て選ぶ。R2P2 は dual CDC のためどちらに繋ぐか要件定義が必要 |
| `reset`/`bootloader` は `exec` のエイリアス | R2P2 だと `Machine.reset` / `Machine.dfu` 相当を IRB で叩く or 専用ショートカット |
| `edit` | IRB / PicoModem で read → tempfile → `$EDITOR` → write |
| `mip install` | 等価なパッケージレジストリが PicoRuby にはまだ無い（将来検討） |

PicoRuby 側で大きく異なるのは、**R2P2 が Unix ライクなシェル** である点と、**PicoModem (RBTP) というバイナリ転送プロトコルが既に存在する** 点。詳細は [`picoruby-r2p2.md`](./picoruby-r2p2.md) を参照。
