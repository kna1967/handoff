# 完了報告: hyena-term 後勝ち接続(-D)＋切断時クリーンアップ(2026-07-11)

## 対象
phase3-repo/webterm/server.pyのみ(【自走指示・確認不要】)。直前の5セッション化タスクの副作用で発覚した、WS切断時のtmuxアタッチ子プロセス残留(ゾンビ化)・新規接続即切断の修正。

## 背景・原因
- `tmux new -A -s <session>`は既存アタッチクライアントを残したまま新規クライアントを追加attachするため、ブラウザのタブ再読込・再接続のたびにtmuxクライアント(pty子プロセス)が積み上がる。
- WS切断時のクリーンアップ(`finally`)が`loop.remove_reader`と`os.close(fd)`のみで、**pty子プロセスへの`os.waitpid()`が一度も呼ばれていなかった**。子プロセス(tmuxクライアント)は自然終了しても親(uvicornサーバー)が回収しないため、defunct(ゾンビ)のまま残留する。
- 実機確認: 修正前の状態でclaudeセッション配下に**40件超のdefunct(tmux: client)** を確認(`ps -eo pid,ppid,stat,cmd`)。

## 実装要点・差分要約(phase3-repo/webterm/server.py)
1. **tmux起動コマンドに`-D`追加**: `tmux new -A -s <session> -c <dir>` → `tmux new -A -D -s <session> -c <dir>`。`-D`はアタッチ時に既存クライアントをデタッチしてから新規クライアントがアタッチする指定で、同一セッションへの再接続・端末乗り換えで常に「後勝ち」になる。
2. **`_terminate_child(pid, grace=1.5)`を新設**: WS切断時のfinallyから呼び出す。forkpty()の子はsetsidで新セッションリーダーになりpid==pgidになる性質を利用し、`os.killpg(pid, SIGTERM)`→0.1秒刻みで`os.waitpid(pid, WNOHANG)`を最大1.5秒ポーリング→未終了なら`os.killpg(pid, SIGKILL)`→`os.waitpid(pid, 0)`で確実に終了・回収する。`ProcessLookupError`/`ChildProcessError`(既に終了・別経路で回収済み)は握りつぶして安全に抜ける。
3. **`pump()`に能動close処理を追加**: pty側がEOFになった(子プロセス終了 or `-D`で他クライアントにデタッチされた)ことを検知した際、`await ws.close()`を能動的に呼ぶよう変更。旧実装ではこの検知後もブラウザ側WebSocketは開いたままで、出力だけ止まって画面が固まる(切断と気付けない)状態になっていた。能動closeにより、index.html側の既存自動再接続ロジックが正しく発火するようになった。
4. 認証方式・`SESSION_CWD`マッピング・許可リスト方式は無変更。

## 確認結果
- **バックアップ**: 変更前に`server.py.bak.2`を作成(既存の`server.py.bak`は前回タスクの巻き戻し用として温存)。`.gitignore`の`*.bak`パターンで追跡対象外。
- **構文チェック**: `python3 -m py_compile server.py` 成功。
- **後勝ち(-D)動作確認**: 未使用セッション`claude5`に2秒差で2本のWS接続を張り、1.5秒(サーバー側grace)経過後を検証。先発 close_code=1000(サーバーが能動closeで正常切断)・後発は生存(close_code=None)を確認(PASS)。
- **残留クライアントゼロ確認**: 上記切断後`tmux list-clients -t claude5`が空(exit 0・出力なし)であることを確認。
- **全6セッション接続確認**: claude/claude2/claude3/claude4/claude5/shellの全てで認証成功・接続確立を再確認(前回タスクの許可リストに影響なし)。
- **ゾンビ再発なし**: 検証バッチ実行後、`ps -eo pid,ppid,stat,cmd | grep defunct`で本タスク由来の新規ゾンビが無いことを確認。修正前に存在した大量のdefunct(40件超)は、本タスク中の2回のサービス再起動で親プロセス終了→init再親化→自動回収され解消済み。無関係な別プロセス由来のゾンビ1件(2026-07-02からの手動テストサーバ、PPID別・本サービス無関係)は対象外のため未処理・報告のみ。
- **実機への影響と自己修復の確認**: 全6セッション接続テストの過程で、本人のiPhoneが実際に接続中だった`claude`セッションを一時的に巻き込み(-Dで検知・数秒切断)したが、journalctlログでindex.html既存の自動再接続ロジックにより数秒後に自動復帰したことを確認(実害なし)。以後の追加テストは非稼働セッション(`claude5`)に限定して実施。
- **サービス再起動・稼働確認**: `hyena_svc.sh restart hyena-term.service`を2回実行(各コード変更の都度)、`systemctl status`で`Active: active (running)`を確認。

## commit hash
- phase3-repo(server.py): `fb86262`(main反映)
- obsidian-vault(セッション引き継ぎ追記): `e1a224e`(main反映)

## ロールバック手順
- server.py変更を戻す場合: `git -C ~/phase3/phase3-repo revert fb86262` の後 `hyena_svc.sh restart hyena-term.service`(または`server.py.bak.2`を`cp`で復元してから再起動)
- セッション引き継ぎの追記のみ戻す場合: `git -C ~/obsidian-vault revert e1a224e`

## 機密の記載
なし(トークン・.env値・Webhook URL等は本報告にも変更内容にも一切含まれない)
