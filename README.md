# BashaXterm

MobaXtermのような高機能sshクライアントに相当することをLinuxでCUI的に実現

## 機能

- ssh接続先の管理
- 簡単なsshファイル転送（OS標準エクスプローラを使う）
- サーバーステータスの表示
- セカンダリドメイン機能（プライマリが繋がらないときに使う、VPNなど）
- 多段ssh接続
- sshポートフォワーディング

## 1. 設定ファイルフォーマット（TOMLベース）

設定は `~/.bashaxterm.toml` に記述する。

```toml
# ~/.bashaxterm.toml

[[hosts]]
alias      = "prod-db"
user       = "alice"
host       = "user@db.example.com:2222"
secondary  = "db-vpn.example.com"      # プライマリが繋がらないときの別経路
proxy      = ["jump1.example.com:22", "jump2.example.com:22"]
forwards   = ["L5432:localhost:5432"]

[[hosts]]
alias      = "web-01"
user       = "deploy"
host       = "web1.internal"
proxy_jump = ["jump-gw.example.com:22"]
forwards   = []
```

### フィールド説明

- `alias`      : 接続を識別する任意の名称
- `host`       : ユーザ名@接続先ホスト名またはIPアドレス:ポート番号（ユーザ名とポート番号は省略可）
- `secondary`  : プライマリ接続失敗時に試すドメイン／ホスト名（省略可）
- `proxy`      : SSH ProxyJump 用のホストリスト（省略可）
- `forwards`   : ポートフォワーディング定義の文字列配列（`-L`/`-R` オプション形式）

## 2. コマンドライン仕様

| サブコマンド                      | 説明                                            |
| --------------------------- | --------------------------------------------- |
| `bashaxterm list`           | 設定ファイルを読み込んで一覧表示（alias／host／状態）               |
| `bashaxterm in <alias>`     | 指定エントリにSSH接続。失敗時は secondary → proxy\_jump を試行 |
| `bashaxterm status [alias]` | 全／指定ホストの到達性（TCP接続チェック）を表示                     |
| `bashaxterm ft <alias>`     | SSHFS でマウントし、`xdg-open` で展開                   |
| `bashaxterm help`           | ヘルプ表示                                         |

- 共通オプション：`-v/--verbose`（詳細ログ）、`-c <file>`（設定ファイル指定）

## 3. 内部モジュール構成

1. **設定パーサ**
   - TOMLライブラリ（tomlc99）で `hosts` テーブルを配列取得し、`HostEntry` 構造体に格納
2. **CLIパーサ**
   - `getopt_long()` でサブコマンドとオプションを解析
3. **SSHコマンドビルダ**
   - `HostEntry` から基本コマンド `ssh -p port user@host` を生成
   - `-J` (ProxyJump) / `-L`/`-R` (フォワード) オプション付与
4. **フォールバックロジック**
   - `in`：プライマリ → secondary → proxy\_jump 経由の順で `execvp()` をリトライ
5. **サーバーステータスチェッカ**
   - POSIXソケットで TCP ポート接続試行 (タイムアウト3秒) し、可否を表示
6. **ファイル転送／マウント**
   - SSHFS で一時ディレクトリ `/tmp/bashax_<alias>` にマウント
   - マウント後に `xdg-open` を呼び出しホストのファイルを表示
7. **ログ & エラー**
   - ANSIエスケープで色付け、`--verbose` で実行コマンドを出力
