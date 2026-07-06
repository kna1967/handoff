# report: sync_alerts_from_notes.py 登録時既成立アラートのreached扱いパッチ(2026-07-06)

## タスク
frontmatterのentry_probe/entry_fullをalerts.dbへupsertする際、登録時点で現在値が既に
ターゲット価格以下だと初回チェックで「今到達した」かのように誤発火・Slack通知される問題への対応。
実例: 6637(entry_probe 3,937円/entry_full 3,700円、分析時点の現在値3,665円)。

## 実装要点
- 対象は指示どおり `sync_alerts_from_notes.py` のみ。`alert_system.py`本体・alerts.dbスキーマ・
  port 8081系は無変更(スキーマ変更は不要だったため報告して止める事態にはならず)。
- 新規関数 `get_latest_close(sec_code)`: `dashboard/data/<sec_code>.json` の `price` フィールドを
  優先し、無ければ `alert_system._jq_client()` / `alert_system._get_latest_close()`(既存の
  J-Quants取得経路をそのまま流用、AdjC優先で分割調整済み)にフォールバック。取得不能なら`None`。
- `build_alert_specs()` に `latest_close` 引数を追加し、entry_probe/entry_full の各specに
  `"reached"`(最新終値が閾値以下か)を付与。
- `sync_one()` の**新規INSERT時のみ**、`reached=True` なら既存カラム
  `last_triggered_at`/`acknowledged_at` に登録時刻を設定して「発火済み・ack済み」の状態で
  登録(スキーマ変更なし)。ログに「登録時既成立のためreached扱いで登録（通知スキップ）」を明記。
  終値取得不能時は従来通り未発火(安全側)。
- **UPDATE(既存auto行の条件/note更新)ではreached再判定を行わない**(既存の発火履歴・ack状態を
  保護するため。スコープを「登録(INSERT)時のみ」に限定)。

## 検証結果
- 事前に6637・1980の既存auto価格アラート(各2件)を削除し、新規INSERTパスを実際に通す形で検証。
- **6637**(現在値3,665円 < entry_probe 3,937円・entry_full 3,700円、条件成立済み):
  `--dry-run`・実行(`--test-slack`)いずれもentry_probe/entry_fullの2件で
  「登録時既成立のためreached扱いで登録（通知スキップ）」ログを確認。実行後、DBで両行の
  `last_triggered_at`/`acknowledged_at`が登録時刻で設定され、`active=1`(ダッシュボードには
  通常どおり表示され「アラート到達」表示になる)であることを確認。
- **1980**(現在値2,833円 > entry_probe 2,557円・entry_full 2,294円、条件未成立):
  同様に新規INSERTを発生させたが、reachedログは出ず、DB上も`last_triggered_at=NULL`のまま
  (従来通り未発火登録)であることを確認。
- 5471等、対象外の既存レコード(48件)への影響がないことを確認(全体のactiveレコード数は
  操作前後で52件のまま、5471のlast_triggered_at等も無変更)。
- `--dry-run`を再実行し、新規0件(冪等)であることを確認。

## 既知の制約(重要・要ケンジ判断)
`alert_system.check_all_alerts()` は `last_triggered_at` を一切参照せず、実行の都度
`judge_price`で終値を再評価して発火要否とSlack通知要否を決める(コード確認・実機検証で確認済み:
`check_all_alerts(notify_slack=False)` を実行したところ、reached登録直後の6637アラート2件が
「発火」として再検出された)。そのため、**本パッチが防ぐのは「登録直後の初回発火」のみ**であり、
条件が成立し続ける限り、以後の`alert_system.py check`(cron)実行でのSlack通知は本パッチ後も
従来どおり発生し得る。恒久的な抑制には`alert_system.py`の`check_all_alerts`側の変更
(例: 状態遷移(未発火→発火)の時だけ通知する等)が必要だが、これは今回の指示範囲外
(`alert_system.py本体は不変更`)のため未実装。対応要否・実装方針はケンジ判断が必要。

## 制約の遵守
- 変更ファイルは `sync_alerts_from_notes.py` のみ。`alert_system.py`・alerts.dbスキーマ・
  port 8081系ダッシュボードコードには一切触れていない
- 検証のため一時的に6637・1980のauto価格アラート計4件をDELETE→再sync実行で再作成したが、
  最終状態は「6637=reached登録・1980=従来通り未発火登録」という意図した結果であり、
  他の対象銘柄・非auto(手動登録)行には一切影響なし
- 機密情報は含まれていない

## commit
- phase3-repo: `9e4dd50`(sync_alerts_from_notes.py reached扱いパッチ)
- obsidian-vault: `e7ffd1e`(セッション引き継ぎ追記)

## ロールバック手順
- `git -C ~/phase3/phase3-repo revert 9e4dd50`
- ロールバック後、DBに残る6637の`last_triggered_at`/`acknowledged_at`(reached状態)は
  データとして残るが実害はない(気になる場合は `UPDATE alerts SET last_triggered_at=NULL,
  acknowledged_at=NULL WHERE sec_code='6637' AND alert_type='price'` で手動クリア可)
