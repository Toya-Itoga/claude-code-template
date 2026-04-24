# CLAUDE.md

## アーキテクチャ
- 

## ディレクトリ構成
- `.venv/`                - 仮想環境
- `src/`                  - アプリ本体
- `src/routers`
- `src/services`
- `src/ripositories`
- `src/templates`
- `config/`               - 設定・ドキュメント
- `scripts/`              - 開発補助スクリプト
- `tests/`                - テストコード
- `.claude/commands/`     - コマンド
- `.claude/agents/`       - サブエージェント定義

## コーディング規約
- コメントを書く
- コードを機能単位のブロックで分割する
- 生成するコードはsrc/に配置すること
- config/のファイルは読み取り専用

## データベース
- 環境ごとにDBを分けること
- .envを参照し切り替えること

## 禁止事項
- .envファイルの参照
- any型の乱用禁止

## 更新ポリシー
- 機能追加時はREADME.mdを更新すること
- AIが間違った動作をしたらここに反映すること

## 技術スタック
- Python
- FastAPI
- uvicorn
- SQLAlchemy
- SQLite
- 