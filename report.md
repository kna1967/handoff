## 2026-07-05 ROIC評価規律の正典化(fetch_stock.py改修+軸⑭マスター新設+テンプレ/SKILL反映)

### 実施内容(rules/corrections.md・mistakes.mdを先に確認済み。判定値の正典一元化・数値創作禁止という既存原則と整合させて実施)

**ステップ1: fetch_stock.py改修(phase3-repo)**
- 財務10年の出力JSON各行に `total_assets`(TA、連結総資産の生値)と `roic_simple`(簡易ROIC%、`round(OP*0.70/TA*100, 1)`)を追加
- 既存の「実データ行のみ採用・予告/修正の空行除外」ロジックはそのまま踏襲(変更なし)
- コード内コメントに「定義の正典はobsidian-vault/銘柄分析/テンプレ/v5_軸⑭_利確規律システム.mdのROIC評価ブロック」と明記
- **検証**: 1980で実行し、前回handoff報告値と完全一致を確認(FY26/3: OP 344.79億・TA 2,320.74億・roic_simple **10.4** / FY25/3: roic_simple **7.5** / FY17/3: roic_simple **4.0**)。不一致なし、停止不要だった

**ステップ2: 軸⑭マスター改訂(obsidian-vault、v5.5→v5.5.5)**
- 「実行記録テンプレート」セクションを丸ごと削除(死蔵判定。執行記録の受け皿はfrontmatter投資管理ブロックのavg_cost設計に統合)
- 「期間ストップ」の1年の行に「ROIC評価ブロックのWACC目安(7%)の点検を含む」を追記
- 「ハイエナ的距離感判定」セクションの直後に「## ROIC評価ブロック(資本効率の判定規律)」を指示文言のまま一字一句挿入(算出定義: NOPAT=営業利益×0.70固定/簡易ROIC=NOPAT÷総資産[傾き用]/正規ROIC=NOPAT÷投下資本[水準・WACC比較用、無借金近似ルール込み]。判定閾値: WACC目安7%固定・水準ガイド7段階・簡易ROICは絶対水準で断定しない等。例外注記・コア⑩との役割分担を含む)
- 改訂履歴に「v5.5.5(2026-07-05)」を追記

**ステップ3: 15軸テンプレへの参照追記(obsidian-vault)**
- v5_15軸ハイエナ流分析テンプレート.mdのコア⑩末尾に、軸⑭マスターのROIC評価ブロックを参照する1ブロックを追記(値の再掲はせず参照型)

**ステップ4: SKILL.mdへの手順追記(obsidian-vault、v5.5.2→v5.5.3)**
- コア⑩の作業手順に「fetch_stock.py出力の財務10年からroic_simple10期表を作成しコア⑩付随ブロックに記載。有利子負債僅少(自己資本の5%未満)を短信BSで確認できれば正規ROIC=NOPAT÷Eqを算出、確認できなければ[要確認]としてEDINET取得に回す」を追加
- 改訂履歴に「v5.5.3(2026-07-05)」を追記

**ステップ5: 終了儀式**
- phase3-repo: `2aa25a7` — `fetch_stock.py: 財務10年出力にTA(総資産)・roic_simple(簡易ROIC)を追加`
- obsidian-vault(軸⑭+テンプレ+SKILL.md): `0a61f7d` — `ROIC評価ブロック新設: 軸⑭マスター正典化+15軸テンプレ参照追記+SKILL.md作業手順追記(v5.5.5/SKILL v5.5.3)`
- obsidian-vault(セッション引き継ぎ.md): `15e2f02` — 本日ブロック+07-04後半の未記録分(7238分析・sync_alerts修正・hyena-pull公開・handoff移行)をまとめて追記
- 編集した3ファイルはすべて事前に`.bak_20260705_roic`を作成済み(git管理外・ローカルのみ)

### ロールバック手順
```bash
# phase3-repo(fetch_stock.py)
git -C ~/phase3/phase3-repo revert 2aa25a7 --no-edit
git -C ~/phase3/phase3-repo push
# または即座に戻したい場合
cp ~/phase3/phase3-repo/fetch_stock.py.bak_20260705_roic ~/phase3/phase3-repo/fetch_stock.py

# obsidian-vault(軸⑭マスター・15軸テンプレ・SKILL.md)
git -C ~/obsidian-vault revert 0a61f7d --no-edit
git -C ~/obsidian-vault push
# または個別ファイルを.bakから即座に戻す
cp "~/obsidian-vault/銘柄分析/テンプレ/v5_軸⑭_利確規律システム.md.bak_20260705_roic" "~/obsidian-vault/銘柄分析/テンプレ/v5_軸⑭_利確規律システム.md"
cp "~/obsidian-vault/銘柄分析/テンプレ/v5_15軸ハイエナ流分析テンプレート.md.bak_20260705_roic" "~/obsidian-vault/銘柄分析/テンプレ/v5_15軸ハイエナ流分析テンプレート.md"
cp ~/obsidian-vault/.claude/skills/v5.3-stock-analysis/SKILL.md.bak_20260705_roic ~/obsidian-vault/.claude/skills/v5.3-stock-analysis/SKILL.md

# セッション引き継ぎ.mdのみ戻す場合
git -C ~/obsidian-vault revert 15e2f02 --no-edit
git -C ~/obsidian-vault push
```

### 今後の運用メモ
- 既存の全分析ノート(1885/1941/1942/6525/7238/5471/1980等)にはROIC評価ブロックは遡及適用しない(SKILL.mdの「既存ノートへの遡及適用」原則どおり、今後の新規分析・再分析から適用)
- 正規ROIC算出には「当期の有利子負債が自己資本の5%未満」の確認が必要(短信BSで都度確認。無借金近似が使えない銘柄はEDINET取得または[要確認]に回す運用)
