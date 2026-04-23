# picoremote 設計メモ

mpremote と PicoRuby/R2P2 の調査を踏まえて、picoremote で採る方針の素案。決定ではなく、議論のたたき台。

## 0. 前提

- ターゲット: R2P2 シェルを搭載した Raspberry Pi Pico / Pico W / Pico 2 / Pico 2 W
- ホスト: macOS / Linux / Windows（まずは macOS/Linux 優先でよい）
- 接続: USB CDC シリアル（115200 8N1）。Dual CDC（メイン + デバッグ）を想定
- 想定ユースケース:
  1. 複数ファイルのデプロイ（プロジェクト一式を Pico に転送）
  2. 手元でコード編集 → 即デバイス上で実行（リモート `run`）
  3. REPL/シェル常駐ターミナル（mpremote の最頻用ユースケース）
  4. ログ取得、CDC1 デバッグ出力の観測
  5. DFU によるファームウェア更新

## 1. 言語と配布

| 候補 | 長所 | 短所 |
|---|---|---|
| Python (mpremote と同じ) | `pyserial` + `platformdirs` が枯れてる / mpremote の知識を流用可 | Ruby エコシステム外から見ると違和感。PicoRuby と Ruby 書く人が多い |
| **Ruby (CRuby gem)** | PicoRuby コミュニティと親和。`bundle install` / `gem install` が自然 | シリアル I/O は `serialport` gem が必要。Windows サポートがやや弱い |
| Go / Rust | 単一バイナリ配布・CI が楽 | ユーザーがビルド環境を持たない場合にありがたいが、貢献障壁は上がる |

**推奨: Ruby（CRuby gem）**。PicoRuby 界隈に導入しやすく、PicoModem のプロトコル定義を `picoruby-picomodem` と共有しやすい（フレーム組み立てロジックを 1 箇所に寄せられる可能性あり）。

## 2. CLI 設計

mpremote の「コマンドチェイン + `+` 区切り + 暗黙の `connect auto`/`repl`」は優秀なので流用する。

```
picoremote                        # auto connect → shell 監視（mpremote の repl 相当）
picoremote connect <device>       # list / auto / id:<sn> / port:<path> / 直接パス
picoremote disconnect
picoremote shell                  # R2P2 シェルに繋ぐ（デフォルト）
picoremote irb                    # 起動後 `irb<CR>` を送って IRB モードに入る
picoremote reset                  # Machine.reset を IRB 経由で叩く
picoremote bootloader             # Machine.dfu(or 相当) を叩く（R2P2 の挙動次第）
picoremote ls [path]              # PicoModem 経由で一覧
picoremote cat <remote>
picoremote cp <src> <dst>         # `:` プレフィックスでリモート区別（mpremote 互換）
picoremote rm [-r] <remote>
picoremote mkdir <remote>
picoremote rmdir <remote>
picoremote run <local.rb>         # PicoModem で /tmp/<hash>.rb へ転送 → IRB で load
picoremote exec '<ruby-code>'     # IRB に流してレスポンスをパース
picoremote eval '<expr>'          # 同上（`p` でラップ）
picoremote edit <remote>          # cat で取得 → $EDITOR → PicoModem で書き戻し
picoremote dfu <file.dfu>         # PicoModem DFU_START 経由でファーム更新
picoremote devs                   # ポート一覧（mpremote と同じエイリアス）
picoremote sleep <ms>             # チェイン中の待機
picoremote log [--cdc1]           # CDC1 のデバッグログを tail
```

### ショートカット

mpremote と同じエイリアスルール（`a0`, `a1`, `u0`, `c0` など + ユーザー定義 `~/.config/picoremote/config.rb`）。

## 3. トランスポート層設計

### 3.1 2 層構成を想定

```
┌─────────────────────────────────────────────┐
│ Commands (ls, cp, run, shell, reset, ...)   │
├─────────────────────────────────────────────┤
│ ShellScraper          │  PicoModemClient    │  ← ユースケースに応じて選択
│ (IRB / shell を叩く)  │  (RBTP フレーム)    │
├─────────────────────────────────────────────┤
│ SerialTransport (CDC0)   +   DebugTap (CDC1)│
└─────────────────────────────────────────────┘
```

- **PicoModemClient**: バイナリ確実性が必要な処理（ファイル転送・DFU）はこっち。`Ctrl-B` でセッション開始、1 フレーム = 1 セッション。
- **ShellScraper**: 軽量な問い合わせ（`ls`, `stat`, `reset` 相当）はシェル/IRB にコマンド送ってテキストパース。どうしても ANSI コントロール等で不安定になりがちなので、**パースが必要な箇所は IRB の `p` で inspect させて JSON 風に取る** のが堅い。

### 3.2 Shell 状態遷移

- 起動直後はシェル（`r2p2>` 相当）
- `irb\n` で IRB へ遷移（プロンプトが `irb>` に変わる）
- IRB 内で `exit` or `Ctrl-D` でシェルに戻る
- どのモードか不明なときは `"\x03"` (Ctrl-C) を打ち込んで入力バッファを空にし、`\n` でプロンプトを釣って判定する

PicoModem セッション中はシェルに入力はリレーされないので、**モード管理ステートマシン**を picoremote 内に持つ必要がある。

### 3.3 エスケープキー

mpremote 互換で抜けキーを `Ctrl-]` / `Ctrl-x` にする。ただし:

- **`Ctrl-B` (0x02) はデバイス側で PicoModem 起動に使う** ため、ユーザー誤操作で勝手にモード遷移しないように、ホスト側の対話ターミナルからは `Ctrl-B` をスルー／警告するオプションを検討
- picoremote 側が明示的に PicoModem を使う場合にのみ Ctrl-B を送る

## 4. リモート `run` の実装候補

1. **PicoModem + `load`**: ローカルのファイルを `/tmp/<hash>.rb` に FILE_WRITE → IRB で `load "/tmp/<hash>.rb"` → 完了後削除。**mpremote の `run` に一番近い挙動**。
2. **IRB ラインずつ流し込み**: 各行を CR 送って IRB プロンプト待ち。サイズが大きいと遅い&複数行構文の扱いが厄介。
3. **シェルに `ruby /tmp/xxx.rb` 風コマンドを作らせる**: R2P2 の `/bin/` にワンショット実行コマンドがあればそれに乗る（現状は IRB が前提）。

**推奨: 1 を基本、小さな `exec`/`eval` は 2**。

## 5. リモート `ls` / `stat` の実装候補

- IRB で `p Dir.entries(<path>)` / `p File::Stat.new(<path>).mode` させてホスト側で `eval` 相当のパースを行う
- mpremote のような `ast.literal_eval` の代わりに、**Ruby の inspect 出力を JSON っぽくホスト側で受ける**（tolerant parser を書くか、または IRB に `require "json"; puts JSON.generate(...)` させる）
- ただし IRB 出力は ANSI カラー/プロンプトが混ざるので、**確実な境界マーカー**（例: `"<<<BEGIN>>>" ... "<<<END>>>"` で挟む）を出力側で仕込む

**代替**: PicoRuby 側に picoremote 用の `/bin/pr-meta` コマンドを用意しておくと、フォーマット固定で楽になる（ただし本体側の変更が必要）。まずはホスト側だけで完結する方式から。

## 6. CDC1（デバッグ出力）対応

- `picoremote log --cdc1` でサブシリアルを tail できる
- `picoremote shell --tap-cdc1` で CDC0 と CDC1 を同時オープンし、CDC1 の出力を `[debug] ...` プレフィックス付きで混合表示する
- 片方しか接続できない環境（prod ビルドなど）は自動でフォールバック

## 7. `reset` / `bootloader`

- R2P2 側の実装確認が必要（`Machine.reset` / `Machine.dfu` / BOOTSEL 相当 API があるか）
- 見つからない場合は **ハードウェアリセット用の BOOTSEL モードにするには物理ボタンが要る** ので、その旨を警告する
- mpremote のように `exec --no-follow "..."` のエイリアスにする方針で統一

## 8. 設定ファイル

- 場所: `~/.config/picoremote/config.rb`（macOS/Linux）/ `%APPDATA%\picoremote\config.rb`（Windows）
- 構文は Ruby（評価して `COMMANDS` のような定数を参照）:
  ```ruby
  COMMANDS = {
    "mypico"   => "connect id:E6612483CB1BAF25",
    "deploy"   => ["cp -r app :", "reset"],
    "demo"     => ["run", "demo/main.rb"],
  }
  ```

## 9. フェーズ分け（実装順の提案）

### MVP (v0.1)

1. `connect` / `disconnect` / `devs` / 自動接続 / `id:`/`port:` 対応
2. インタラクティブ `shell`（CDC0 の生シリアルモニタ + Ctrl-] で抜ける）
3. `cp` ローカル → リモート（PicoModem FILE_WRITE）
4. `cp` リモート → ローカル（PicoModem FILE_READ）
5. `reset`（IRB 経由）

### v0.2

6. `ls` / `cat` / `rm` / `mkdir`（シェルの既存コマンド経由 + パース）
7. `run <local.rb>`（PicoModem + IRB `load`）
8. `eval` / `exec`（IRB 行送り + マーカー付き結果取得）
9. `log --cdc1`（副 CDC の tail）
10. `edit`（round-trip）

### v0.3

11. `dfu`（PicoModem DFU_START）
12. ユーザー設定ファイル（`config.rb`）
13. コマンドチェイン（`+` 区切り）

### v0.4 以降（任意）

14. `mount` 相当（R2P2 側に RemoteFS を注入するアプローチ。大工事）
15. パッケージ管理（PicoRuby 向け mip 相当が整備された後）

## 10. 未決事項 / TODO

- [ ] 実機の USB VID/PID を確認して `auto` の判定条件を確定する
- [ ] R2P2 側に `Machine.reset` / `Machine.bootloader` 相当 API があるか要確認
- [ ] IRB のプロンプト文字列仕様（プレフィックス、複数行継続時の `* `）を実機で採取
- [ ] `Ctrl-B` をホスト側のエスケープと衝突させない UX の決定
- [ ] `picoruby-picomodem` と共有する RBTP 定義を Ruby モジュールとして切り出せるか検討
- [ ] RSpec / minitest ベースの統合テスト（ループバックまたは QEMU + rp2040js）を構築
- [ ] パッケージ名は `picoremote` でよいか（`pico-remote` 等の類似 gem/pypi パッケージ確認）
