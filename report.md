## 2026-07-04 hyena-pull をGitHub公開・BRATインストール可能化

### 実施内容
1. **機密チェック(push前に実施)**: `grep -rniE "ghp_|github_pat_|token=|slack|webhook|api[_-]?key|secret|password=|IPアドレスパターン|tailscale|jquants|edinet|/home/ubuntu"` 等でソース全体を確認。実害ある一致はゼロ。唯一の一致は`main.ts`のUIプレースホルダ文字列`"ghp_..."`(トークン入力欄のヒント表示用、実トークンではない)。ユーザー名`kna1967`はauthor欄とowner欄のプレースホルダ例として3箇所に出現(公開GitHubユーザー名であり非機密)。package-lock.jsonは通常のnpmレジストリ参照のみ(40件)。以上を確認した上でpushを実行
2. **リポジトリ作成+push**: `gh repo create kna1967/hyena-pull --public --source . --push` 成功。https://github.com/kna1967/hyena-pull
3. **リリース作成**: manifest.jsonのversion(0.1.0)と一致するタグでリリース作成。`gh release create 0.1.0 main.js manifest.json --title "hyena-pull 0.1.0" --notes "pull-only GitHub sync for Obsidian"` 成功。アセット2点(main.js/manifest.json)を確認済み。BRAT用のリポジトリ構成として問題なし(リリースにmain.js/manifest.jsonが揃っている)

### 確認結果
- `gh release view 0.1.0`でasset: main.js / manifest.jsonの2点を確認
- リポジトリURL: https://github.com/kna1967/hyena-pull
- リリースURL: https://github.com/kna1967/hyena-pull/releases/tag/0.1.0
- ローカルリポジトリの`main`ブランチがorigin/mainを追跡する状態に設定済み

### commit hash
前回報告のローカルcommit `ba63de9` がそのままリモートのmainブランチ先端としてpush済み(コミット内容の変更なし、公開のみ実施)

### ロールバック手順(必要な場合)
```bash
gh release delete 0.1.0 -R kna1967/hyena-pull -y
gh repo delete kna1967/hyena-pull --yes
```
(ローカルの `~/phase3/obsidian-hyena-pull/` はそのまま残る。origin削除は `git -C ~/phase3/obsidian-hyena-pull remote remove origin`)

