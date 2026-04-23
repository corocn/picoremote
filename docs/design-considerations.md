# picoremote 設計メモ

mpremote と PicoRuby/R2P2 の調査を踏まえて、picoremote で採る方針の素案。**「なぜそうするか／どう作るか」の選択論** をまとめる。

実装順や具体的なフェーズ分けは [`roadmap.md`](./roadmap.md) を参照。

## 0. 前提

- ターゲット: R2P2 シェルを搭載した Raspberry Pi Pico / Pico W / Pico 2 / Pico 2 W (RP2040 / RP2350)
- ホスト: macOS / Linux / Windows（まずは macOS / Linux 優先）
- 接続: USB CDC シリアル（115200 8N1）。Dual CDC（メイン + デバッグ）を想定
- 想定ユースケース（優先度順）:
  1. **手元の `.rb` をワンショットで Pico に流して実行**（MVP の主目的）
  2. プロジェクト一式のデプロイ（複数ファイル転送）
  3. REPL/シェル常駐ターミナル
  4. CDC1 デバッグ出力の観測
  5. DFU によるファーム更新

## 1. 言語と配布

| 候補 | 長所 | 短所 |
|---|---|---|
| Python (mpremote と同じ) | `pyserial` + `platformdirs` が枯れてる / mpremote の知識を流用可 | Ruby エコシステム外から見ると違和感 |
| **Ruby (CRuby gem)** | PicoRuby コミュニティと親和。`gem install` が自然 | シリアル I/O は `serialport` gem が必要。Windows サポートはやや弱い |
| Go / Rust | 単一バイナリ配布・CI が楽 | 貢献障壁は上がる |

**採用: Ruby (CRuby gem)**。PicoRuby 界隈に導入しやすく、`picoruby-picomodem` と RBTP プロトコル定義を共有しやすい（フレーム組み立てロジックを 1 箇所に寄せられる可能性あり）。

## 2. CLI 設計

mpremote の「コマンドチェイン + `+` 区切り + 暗黙の `connect auto`」は便利なので、そのうち取り入れたい。ただし MVP では単発コマンドで十分。

最終的に目指すコマンド一覧（時期は roadmap 参照）:

```
picoremote                        # 引数なし: shell or 直近の振る舞い
picoremote connect <device>       # list / auto / id:<sn> / port:<path> / 直接パス
picoremote disconnect
picoremote devs                   # ポート一覧

picoremote run <local.rb>         # PicoModem で /tmp/<hash>.rb 転送 → IRB で load
picoremote shell                  # CDC0 の対話シェル
picoremote irb                    # 起動後 `irb` を送って IRB モードへ
picoremote reset                  # Machine.reset を IRB 経由で叩く
picoremote bootloader             # DFU/BOOTSEL モードへ
picoremote log [--cdc1]           # CDC1 のデバッグログを tail

picoremote ls [path]
picoremote cat <remote>
picoremote cp <src> <dst>         # `:` プレフィックスでリモート区別 (mpremote 互換)
picoremote rm [-r] <remote>
picoremote mkdir <remote>
picoremote rmdir <remote>
picoremote edit <remote>          # cat → $EDITOR → 書き戻し

picoremote eval '<expr>'
picoremote exec '<ruby-code>'

picoremote dfu <file.dfu>
picoremote sleep <ms>             # チェイン中の待機
```

### ショートカット

mpremote と同じエイリアスルールを目指す（`a0`, `a1`, `u0`, `c0` … + ユーザー定義 `~/.config/picoremote/config.rb`）。

## 3. トランスポート層設計

### 3.1 2 層構成

```
┌─────────────────────────────────────────────┐
│ Commands (run, ls, cp, shell, reset, ...)   │
├─────────────────────────────────────────────┤
│ ShellDriver           │  PicoModemClient    │  ← ユースケースに応じて選択
│ (IRB / shell を叩く)  │  (RBTP フレーム)    │
├─────────────────────────────────────────────┤
│ SerialTransport (CDC0)   +   DebugTap (CDC1)│
└─────────────────────────────────────────────┘
```

- **PicoModemClient**: バイナリ確実性が必要な処理（ファイル転送・DFU）はこちら。`Ctrl-B` でセッション開始、1 フレーム = 1 セッション。
- **ShellDriver**: 軽量な問い合わせ・実行（`run`, `eval`, `reset` 相当）はシェル/IRB にコマンド送ってテキストパース。

### 3.2 シェル状態遷移

R2P2 には Raw REPL のような機械向けモードがないので、**ホスト側でシェル状態を持つ必要がある**。

- 起動直後はシェル（プロンプト要採取）
- `irb\n` で IRB へ遷移（プロンプト要採取）
- IRB 内で `exit` or `Ctrl-D` でシェルに戻る
- どのモードか不明なときは `\x03` (Ctrl-C) で入力バッファを空にし、`\n` でプロンプトを釣って判定

PicoModem セッション中はシェルに入力がリレーされないので、**モード管理ステートマシン**を picoremote 内に持つ。

### 3.3 エスケープキー

mpremote 互換で抜けキーを `Ctrl-]` / `Ctrl-x` にする予定。ただし:

- **`Ctrl-B` (0x02) はデバイス側で PicoModem 起動に使う**
- 対話ターミナル中にユーザーが誤って `Ctrl-B` を打つと意図せず PicoModem セッションに入ってしまう
- ホスト側でデフォルト抑止 + フラグで通すなどの設計が要る

## 4. リモート `run` の実装

候補:

1. **PicoModem + `load`**（採用予定）
   - ローカルファイルを `/tmp/<hash>.rb` に FILE_WRITE
   - シェル → IRB に切り替え、`load "/tmp/<hash>.rb"` を 1 行送信
   - IRB プロンプトが戻るまで stdout を流す
   - 終わったら IRB を抜けて `/tmp/<hash>.rb` を削除（オプション）
   - **mpremote の `run` に一番近い挙動**
2. **IRB ラインずつ流し込み**
   - 各行を CR 送って IRB プロンプトを待つ
   - サイズが大きいと遅い、複数行構文（`def`/`class`/`do`/ヒアドキュメント）の扱いが厄介
   - → 小さな `exec`/`eval` 用途に限定
3. **シェルに `ruby /tmp/xxx.rb` を打たせる**
   - R2P2 の `/bin/` にワンショット実行コマンドがあれば乗れるが、現状は IRB が前提

**結論: 1 を主軸、2 は `eval`/`exec` 専用**。

### 終了検知

`load` 完了の検知が肝。候補:

- IRB プロンプトの再出現を待つ（プロンプト文字列の正確な仕様確認が必要）
- 送信前後で **境界マーカー**（`puts "<<<PR-END>>>"` など）を仕込んで検出する
- 例外発生時は `=> #<Exception ...>` の inspect が出るので、それも拾う

## 5. リモート `ls` / `stat` の実装

- IRB に `p Dir.entries(<path>)` / `p File::Stat.new(<path>).mode` を送ってホスト側でパース
- mpremote のような `ast.literal_eval` の代わりに、**Ruby の inspect 出力をホスト側のトレラントパーサで受ける**、または IRB に `require "json"; puts JSON.generate(...)` を送る（PicoRuby に json があるか要確認）
- ANSI カラー / プロンプトが混ざるので、**境界マーカー**で囲うのが堅実

**代替**: PicoRuby 側に picoremote 用の `/bin/pr-meta` 的ヘルパーを用意するとフォーマット固定化できて楽。ただし本体側の変更が必要なので、まずはホスト側だけで完結する方式を試す。

## 6. CDC1（デバッグ出力）対応

- `picoremote log --cdc1` でサブシリアルを単独 tail
- `picoremote shell --tap-cdc1` で CDC0 と CDC1 を同時オープン、CDC1 の出力を `[debug] ...` プレフィックス付きで混合表示
- prod ビルドでは CDC1 が無いことがあるので、open 失敗時は静かにフォールバック

## 7. `reset` / `bootloader`

- R2P2 側の API 確認待ち（`Machine.reset` / `Machine.bootloader` 相当が存在するか）
- 無い場合は **BOOTSEL モードに入るには物理ボタンが必要**な旨を警告メッセージで案内
- 実装方針は mpremote と同じく `exec --no-follow "..."` のエイリアス扱いで統一

## 8. 設定ファイル

- 場所: `~/.config/picoremote/config.rb`（macOS/Linux）/ `%APPDATA%\picoremote\config.rb`（Windows）
- 構文は Ruby:
  ```ruby
  COMMANDS = {
    "mypico"   => "connect id:E6612483CB1BAF25",
    "deploy"   => ["cp -r app :", "reset"],
    "demo"     => ["run", "demo/main.rb"],
  }
  ```

## 9. 設計上の未決事項

実装ブロッカー（実機調査・仕様確認）は [`roadmap.md`](./roadmap.md) の TODO に分けて書く。ここに残すのは **設計判断そのもの** のうち、まだ結論を出していないもの。

- **PicoModem プロトコル定義の共有方法**: `picoruby-picomodem` の RBTP 定義を Ruby gem として切り出して、デバイス側 / ホスト側 (picoremote) の両方から `require` できるようにすべきか？ それともコピー実装で割り切るか
- **`Ctrl-B` 衝突の UX**: 対話シェル中に Ctrl-B を打ったとき、(a) 黙ってスルー (b) 警告を出す (c) 確認後に通す のどれをデフォルトにするか
- **境界マーカー方式 vs プロンプト検出方式**: ShellDriver の終了検知をどちらに寄せるか。マーカー方式は安定だが、ユーザー側の `.rb` がそのマーカー文字列を含んでいる事故が起きうる
- **inspect → JSON 変換**: PicoRuby に `json` gem があるか／無ければ `inspect` のトレラントパーサを書く必要があるか
- **gem 名の重複確認**: rubygems.org に `picoremote` / `pico-remote` / `pico_remote` が既にいないか
- **テスト戦略**: シリアルループバック / rp2040js (qemu) / 実機モック のどれを CI で回すか
