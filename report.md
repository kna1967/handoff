# 完了報告: fetch_kessan.py IRBANK経由フォールバックの追加(2026-07-12)

## 対象
phase3-repo/fetch_kessan.py に、TDnet縦覧期間(約1ヶ月)切れの決算短信を補うIRBANK経由フォールバックを追加。既存のTDnet走査ロジック・cron挙動(tdnet_fetch.py側)は無変更。

## 背景
TDnetの公開保持期間は直近約1ヶ月のみのため、本決算から数ヶ月後に分析する銘柄(実例: 4040南海化学の2026-05-14本決算、7740でも同型)で毎回「決算資料フォルダなし」が発生していた構造的な穴を埋める。

## 実装内容
1. **発動条件**: 既存のTDnet走査が0件だった場合のみ発動。`--no-fallback`または環境変数`FETCH_KESSAN_NO_IRBANK_FALLBACK`で無効化可能(一括バッチからの誤発動防止ガード。本ツール自体は単一銘柄オンデマンド用途のみを想定)。
2. **取得手順**: IRBANK(`https://irbank.net/{4桁コード}`)の「決算発表資料」セクションから「決算短信」を含むリンクを新しい順に最大3件抽出→各詳細ページを経由してPDF実体URL(IRBANK自ミラーf.irbank.net、または外部配信CGI)を特定→Content-Type確認の上で取得。
3. **保存先・命名・冪等性**: 既存のTDnet取得分と完全に同一の規約(`/home/ubuntu/決算資料/{code}/`、同名ファイルは上書きせずスキップ)。
4. **礼儀・負荷の規律**: User-Agentに連絡先(kenzinaka@gmail.com)を明示。robots.txtを毎回確認しDisallow該当時は中止。全リクエスト間に2秒以上のsleep。
5. **失敗時の扱い**: ページ構造変化・PDF不在・コード不明はいずれも例外を投げず「フォールバックも0件」と報告して正常終了(分析フロー側の一次資料ゲートが従来どおり判断する)。

## 仕様との差異(発見・報告)
- 当初想定のリクエスト予算「一覧1+PDF3=4」は、IRBANKの実サイト構造(会社ページ→決算短信ごとの詳細ページ→実PDF、の2段階)を踏まえると成立しないことが判明。決算短信のPDFリンクは詳細ページ内にのみ存在し、一覧ページには直接出てこないため、正しくは「一覧1+候補ごとに(詳細ページ1+PDF1)×最大3件=最大7リクエスト」となる。数値を偽って報告するのではなく、実測に基づき7リクエストへ見直した上で全リクエスト間2秒sleepの礼儀規律は維持した。
- 「外部配信CGI(daiwair.co.jp等)」については、実際にサンプリングした6銘柄(4040/7740/6637/1450/2980/6016)すべてでIRBANK自ミラー(f.irbank.net)への直リンクだった。daiwair.co.jp等への転送は今回のサンプルでは一度も出現しなかったが、パーサーは両パターンに対応済み(直PDFリンク優先、無ければdaiwair.co.jpパターンを探索)。

## 検証結果
- **4040(TDnet 0件→フォールバック発動)**: 実際に本決算・3Q・2Q決算短信PDF計3件を取得(既存の30日走査は0件、フォールバックで全3件保存成功)。取得PDF(2026-05-14本決算)の1ページ目を`pdftotext`で確認し「南海化学株式会社」「2026年３月期」を含むことを確認。
- **3498(TDnetで取れる銘柄)**: 直近開示(2026-07-03)がTDnet走査でヒットし、フォールバックのログが一切出力されないこと(発動しないこと)を確認。
- **--no-fallback(4040で再実行)**: 「TDnetで0件」の後「--no-fallback指定のためIRBANKフォールバックは行いません」と表示され、PDF取得を一切行わない従来動作と一致することを確認。
- **実在しないコード(9999)**: TDnet 0件→フォールバック発動→IRBANK会社ページは200だが「決算発表資料」セクション自体が存在せず0件と正しく判定→例外を投げず「フォールバックも0件でした」で正常終了(exit 0)することを確認。
- robots.txt(`https://irbank.net/robots.txt`)は`Allow: /`(Disallowは`/search`のみ)を事前確認済み。今回アクセスする`/{code}`パスはすべて許可対象。

## 制約の遵守
- 変更対象はfetch_kessan.pyのみ(tdnet_fetch.py本体・cron設定・決算資料フォルダの既存構成は無変更)。既存のTDnet走査ロジックの差分はゼロ(diffで確認済み、追加のみ)。
- IRBANKへのアクセスは1銘柄あたり最大7リクエスト(実測ベースに見直し、上記参照)、全リクエスト間2秒以上のsleep。
- 一括バッチ発動防止は`--no-fallback`とデフォルト無効ではなく、環境変数`FETCH_KESSAN_NO_IRBANK_FALLBACK`のいずれかで担保(将来複数銘柄を回すラッパーを書く場合はこのいずれかを設定する運用とする)。
- パースできない値・見つからないPDFは推定・按分せず、素直に「0件」「フォールバックも0件」と報告する設計を徹底。

## commit
- phase3-repo(fetch_kessan.py): `f90fa27`

---

# 完了報告: cron 30行目修復・CLI v2.1.220/Opus 5 化(2026-07-25)

## 対象
crontab 30行目の構文破損を修復し、19:00のアラートチェックを復旧。あわせて Claude Code CLI を更新。コード変更なし(phase3-repo は無変更)。

## 実施(cron修復)
- crontab 30行目が改行欠落で2エントリ連結していた
- `run_alert_check.sh` と後続の `20 18 * * 1-5 ...` が `run_alert_check.sh20` という不正コマンド名になっていた
- このため19:00のアラートチェックが実行されていなかった
- cron.log に `/bin/sh: 1: //home/ubuntu/phase3/run_alert_check.sh20: not found` が15回記録。最終エラーは07-24 19:00
- 修復方法: nano編集は1行目にゴミが混入して bad minute で失敗した
- 正解手順は `crontab -l` のバックアップを `sed '30c\...'` で加工し `crontab cron.new` で適用する方式
- 修復後の30行目: `0 19 * * 1-5 /home/ubuntu/phase3/run_alert_check.sh >> /home/ubuntu/phase3/cron.log 2>&1`
- 手動実行は exit=0 だがログに痕跡なし(フラグで即抜けたと推測)
- 真の確認は2026-07-26以降の19:00自動実行を待つ

## 訂正
- 07-20(月)の screening 出力欠落は cron 障害ではなかった
- 実際は海の日の祝日スキップ(screening.py の休場ガード)であり、設計どおりの正常動作
- 根拠は cron.log の `[2026-07-20] 市場休場のためスキップ` の1行
- 当初「cron失敗の可能性」と判断したのは誤り

## 環境更新
- Claude Code を v2.1.198 → v2.1.220 に更新
- Opus 5 が /model ピッカーに出現(バージョン不足が原因だった)
- 現在のC1: Opus 5 / xhigh effort / Claude Max
- Opus 5 では effort のリセットが起きない(xhigh が引き継がれた)
- Fable 5 が Max プラン標準化。週次上限の最大50%まで、Opus 4.8 より消費が速い
- claude.ai 側は Opus 5 が選択可能。CLI との表示ラグは `claude update` で解消する

## 新規PENDING(未着手)
- screening の GitHub push が常態的に失敗している
- cron.log に `! [rejected] main -> main (fetch first)` が出ている
- commit は成功するが push で弾かれ、後から手動マージで回収されている
- 証跡: git log の「origin/main マージ」が 07-19・07-25 に出現
- Slack には「送信完了」と表示されるため失敗が見えない
- 推定原因: push 前に git pull していない
- cron.log のエラー(`not found` / `Traceback` / `rejected`)を検知して通知する仕組みが無い
- 今回の15営業日の障害は人間が git log を眺めて偶然発見した

## ロールバック手順
- cron を戻す場合は修復前に取得した `crontab -l` のバックアップを `crontab <backup>` で再適用する
- ただし戻すと19:00アラートチェックが再び動かなくなるため、通常は不要

## commit
- obsidian-vault(セッション引き継ぎ.md): `974b320`

---

# 完了報告: screening の push前pull追加とpush失敗のSlack通知(2026-08-01)

## 対象
phase3-repo/screening.py のみ。crontab・他のcronスクリプト・dashboard等は無変更。

## 背景
- screening の定期実行が git push で常態的にrejectされていた
- cron.log に `! [rejected] main -> main (fetch first)` が記録される一方、Slackには「送信完了」と表示されていた
- 原因はpush前にpullしていないこと。obsidian-vaultにはIRBANK sync・分析PRマージ等の他系統pushが日常的に入るためrejectは必然

## 対象箇所の特定
- crontab 24行目から辿った結果、git処理は呼び出し先シェルではなく screening.py 内にあった
- 変更前は screening.py:305-307 の `subprocess.run(["git", ...])` 3連発（add / commit / push）
- 3つとも戻り値を一切見ておらず、直後の `print("Obsidian vault保存完了")` が無条件だったのが誤通知の原因

## 実装内容
1. `sync_vault_to_github(vault_dir, commit_msg)` を新設（screening.py:18〜、`latest_business_day` の直前）
2. 処理順を add → commit → `git pull --rebase` → push に変更（pullをpushの直前に挿入）
3. 各gitコマンドは `capture_output=True` で受けたうえで stdout/stderr をそのまま print し、cron.log の情報量を落とさない
4. commitは変更なし時にrc!=0になるが正常系として扱い、後続のpull/pushを継続する
5. pull --rebase が失敗した場合、`.git/rebase-merge` `.git/rebase-apply` の有無で rebase進行中かを判定する
6. rebase進行中なら `git rebase --abort` で中断し、pushせずに `(False, 理由)` を返す（強制解決・force pushは一切しない）
7. 戻り値 `(push_ok, detail)` を呼び出し側に伝播

## 終了ステータスの伝播
- 成功時: `Obsidian vault保存完了（GitHub反映済み）: {md_path}`
- 失敗時: `Obsidian vault保存はローカルcommitのみ・GitHub未反映: {md_path} / {detail}`
- Slack成功時: 従来どおり `📊 *v5.2 1次スクリーニング結果 ...*`
- Slack失敗時: `⚠️ *... （GitHub未反映）*` + 理由 + 「ローカルcommitのみ。手動で pull --rebase → push が必要。」
- あわせて Slack POST 自体の応答も判定し、非2xxなら `Slack通知送信失敗: HTTP {code}` を出す（同じ「失敗を成功と表示する」バグのため）

## 検証結果
- 2026-08-01は土曜のため screening.py の休場ガード（`is_business_day`）が先に効き、手動実行では git ブロックまで到達しない
- 実行結果は `[2026-08-01] 市場休場のためスキップ` / exit 0 で、end-to-endの通常フロー実行は本日実施できていない
- 代替として、出荷される `sync_vault_to_github` の定義をASTで screening.py から抽出してexecし、ハンドコピーではなく実物の関数を直接検証した
- テストA(正常系): 実物のobsidian-vaultに対して実行し `push_ok=True`。nothing to commitでもpull/pushまで到達、vaultは汚れず（status --porcelain 空）
- テストB(reject再現): bare remote＋2クローンで他系統が先にpushした状況を作り、`push_ok=True`。remoteのlogに `screening 2026-08-01` と `irbank sync` が両方載り、他系統のcommitを消していないことを確認
- テストC(コンフリクト): 同一ファイルの add/add 衝突を作り `push_ok=False`。`rebase --abort` 実行後に rebase-merge/rebase-apply の残留なし、ローカルcommitは保持、remoteは未更新であることを確認
- テストD: push_ok による Slack 文面分岐が成功用/失敗用に切り替わることを確認
- `py_compile` によるsyntaxチェック通過
- 次回の実地確認は 2026-08-03(月) 07:00 の自動実行。cron.log に `Obsidian vault保存完了（GitHub反映済み）` が出れば成功

## 制約の遵守
- crontabは未編集
- force push なし・rebaseの強制解決なし
- 変更は screening.py のみ。commit時に混在していた dashboard/manual_links.json・master.json・portfolio.json の既存差分は巻き込んでいない
- /home/ubuntu/phase3/screening.py は phase3-repo/screening.py へのシンボリックリンクのため、別途の配置作業は不要

## ロールバック手順
- `git -C /home/ubuntu/phase3/phase3-repo revert 643fd95` で変更前の add/commit/push 3連発に戻る
- ただし戻すとrejectの常態化と誤通知が再発する

## commit
- phase3-repo(screening.py): `643fd95`
