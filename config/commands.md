# commands.md

## Claude Code
- 起動: `claude`
- 再ログイン: `/login`
- 終了: `/exit`
- 直前の操作を取り消す: `/undo`

## Git初期設定

### テンプレートから始める場合
- リモート削除: `git remote remove origin`
- リモート紐付け: `git remote add origin https://github.com/ユーザー名/新しいリポジトリ名.git`
- 最初のpush: `git push -u origin main`

### ゼロから始める場合
- Gitの初期化: `git init`
- リモートの紐付け: `git remote add origin https://github.com/ユーザー名/リポジトリ名.git`
- `git add .`
- `git commit -m "first commit"`
- `git branch -M main`
- `git push -u origin main`

## Git関係
- プッシュ: `sh scripts/push.sh ""`
- 変更履歴確認: `git log --oneline`
- 直前のコミットに戻す: `git revert HEAD`

## サーバー起動
- 開発サーバー: `python -m uvicorn src.main:app --reload`

## テスト
- 全テスト実行: `pytest tests/ -v`

## 仮想環境
- 作成: `python -m venv .venv`
- 有効化: `source .venv/bin/activate`