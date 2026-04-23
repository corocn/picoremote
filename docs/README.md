# picoremote ドキュメント

PicoRuby (R2P2) 向けのデプロイ/REPL/デバッグ支援ツール `picoremote` の設計ノートおよび調査資料を置く場所。

現在は先行事例調査フェーズ。参考にしている mpremote を中心に、そこから得た知見と PicoRuby 特有の事情をまとめている。

## ファイル一覧

- [`mpremote.md`](./mpremote.md) — mpremote の機能・CLI・アーキテクチャ・プロトコル（Raw REPL, Raw Paste, FS Hook）
- [`picoruby-r2p2.md`](./picoruby-r2p2.md) — PicoRuby / R2P2 のシェル構造、USB CDC、PicoModem (RBTP) プロトコル
- [`design-considerations.md`](./design-considerations.md) — picoremote の設計方針メモ（mpremote との差分、採用/見送り判断）

## 参考リンク

- mpremote 公式ドキュメント: <https://docs.micropython.org/en/latest/reference/mpremote.html>
- mpremote ソース: <https://github.com/micropython/micropython/tree/master/tools/mpremote>
- PicoRuby: <https://github.com/picoruby/picoruby>
- R2P2 (旧リポジトリ / 現在は picoruby 配下): <https://github.com/picoruby/picoruby/tree/master/mrbgems/picoruby-r2p2>
- picoruby-picomodem: <https://github.com/picoruby/picoruby/tree/master/mrbgems/picoruby-picomodem>
