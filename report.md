# report: アラート通知の社名フォールバック＋6525社名データ点検(2026-07-06)

## タスクA: 原因確認(結果)

1. **1941・1942の社名空欄**: master.json(40銘柄)・youtube_master.json(174銘柄、修正前)の
   いずれにも1941・1942は不在。両銘柄は6/21のv5.5検証再分析でノート自体は存在するが、
   ウォッチリスト(YouTube動画・手動マスタ)対象外のため社名が引けない状態だった。
2. **6525が「ELECTRIC」表示**: youtube_master.jsonには6525が格納されていたが、値自体が
   誤り("ELECTRIC")だった。原因はyoutube_fetch.pyの`NAME_BEFORE`正規表現。実際の動画タイトル
   「KOKUSAI ELECTRIC(6525)AIの覇者か？...」に対し、正規表現の社名捕捉部分`[^\s...]+`が
   空白を跨げないため、コード直前の1語("ELECTRIC")のみを捕捉し「KOKUSAI」を取りこぼしていた
   (抽出ロジックのバグ。格納値の問題ではなく生成ロジックの問題)。

## タスクB: 社名フォールバック(sync_alerts_from_notes.py)

master∪youtube_master(alert_system.get_stock_name)で引けない場合、ノートファイル名規約
「コード_社名_YYYY-MM-DD_....」から社名部分を抽出するフォールバック`resolve_stock_name()`を
追加。社名自体に"_"を含む場合(例: `6525_KOKUSAI_ELECTRIC_...`)にも対応するため、コード(先頭)の
次からYYYY-MM-DD形式が最初に現れる直前までを社名とみなす方式にした。ファイル名からも取れない
場合は空欄のまま(創作しない)。フォールバック使用時は`[INFO] [コード] 社名がmaster/youtube_master
で引けず、ノートファイル名から補完: ...`とログ出力。既存の個別銘柄ノート74ファイル全件で
抽出成功(失敗0件)を確認済み。

## タスクC: 6525社名の修正(youtube_fetch.py)

原因が抽出ロジックのバグと判明したため、`NAME_BEFORE`正規表現を「半角/全角スペース区切りで
最大3語までの複数語社名」に対応するよう修正(生成側の修正)。

**回帰確認**: 実データ(youtube.json、204本の動画タイトル)全件に対し、旧正規表現と新正規表現の
抽出結果を突合。差分は3件のみで、いずれも意図した正しい修正だった:
- 6525: `ELECTRIC` → `KOKUSAI ELECTRIC`(報告対象のバグ)
- 3993: `Technology` → `PKSHA Technology`(同種のバグ、副次的に発見・修正)
- 3922: `TIMES` → `PR TIMES`(同種のバグ、副次的に発見・修正)
それ以外の172件(全202件中)は完全一致で回帰なし。さらに、現在のyoutube_master.json(175銘柄)を
旧ロジックで再構築したものと実際に regenerate した新ロジックの結果を全銘柄で突合し、
同じく上記3件のみの差分であることを二重に確認した。

`youtube_fetch.fetch_all()`を実際に実行(YouTube API経由、既存の生成経路をそのまま使用)し、
youtube_master.jsonを最新化。6525が正しく`"KOKUSAI ELECTRIC"`になったことを確認。
(この修正で3993・3922も副次的に正しい社名になっている)

## 検証結果

- `sync_alerts_from_notes.py --dry-run`で以下を確認:
  - `[1941]` → `中電工`(ファイル名フォールバック経由、ログにフォールバック使用を明記)
  - `[1942]` → `関電工`(同上)
  - `[6525]` → `KOKUSAI ELECTRIC`(youtube_fetch.py修正後のmaster経由、フォールバック不要)
- Slack実送信は行わず(dry-runで全銘柄「変更なし」だったため実質的な変更報告がなく、
  `resolve_stock_name()`を直接呼び出してフォーマットが正しいことを確認する方法で代替。
  指示の「フォーマット確認できれば送信なしでも可」に従った)。
- alerts.db: 操作前後で全体行数90件・active件数52件に変化なし(社名解決はSlackメッセージの
  表示にのみ影響し、条件・発火状態[last_triggered_at等]には一切触れていないことを確認)。

## 制約の遵守
- 変更ファイルは`sync_alerts_from_notes.py`と`dashboard/youtube_fetch.py`(社名データの
  生成側)のみ。`alert_system.py`・alerts.dbスキーマ・port 8081系ダッシュボードコードには
  一切触れていない
- 機密情報は含まれていない

## commit
- phase3-repo: `6394229`(youtube_fetch.py正規表現修正 + sync_alerts_from_notes.py社名
  フォールバック追加)
- obsidian-vault: `c168f98`(セッション引き継ぎ追記)

## ロールバック手順
`git -C ~/phase3/phase3-repo revert 6394229`。ただし youtube_master.json / youtube.json は
.gitignore対象(自動生成物)のため、revert後も次回cron実行(build_data.py経由)まで修正前の
コードで再生成されない限り値は残る。手動で直近版に戻したい場合は
`/home/ubuntu/phase3/venv/bin/python -c "import sys; sys.path.insert(0,'dashboard'); import youtube_fetch; youtube_fetch.fetch_all()"`
をrevert後に再実行すること(旧ロジックでの再生成)。
