# 開発Tips集 — FastAPI + AWS + Claude Code AIDD

このドキュメントは過去の開発経験から得た知見をまとめたものです。
新しいチャットでこのファイルを共有することで、同等のコンテキストを素早く引き継げます。

---

## 1. プロジェクト構成

### AIDDパイプライン
```
Claude.ai（設計・壁打ち・エラー原因特定）
        ↓
Claude Design（画面デザイン・HANDOFF.md生成）
        ↓
Claude Code（実装・コーディング）
```

### テンプレートリポジトリ
- GitHub: `claude-code-template`（Template repository機能を有効化）
- 新規プロジェクト作成時: `git clone <template> .` → `git remote remove origin`
- CLAUDE.mdは35行以内を目安に簡潔化

### ディレクトリ構成
```
src/
├── main.py
├── routers/        # ルータ層
├── services/       # サービス層
├── repositories/   # リポジトリ層
├── templates/      # Jinja2テンプレート
│   ├── components/ # 共通コンポーネント
│   └── partials/   # HTMXフラグメント
├── static/         # CSS・JS
├── utils/          # ユーティリティ
└── database.py     # DB接続設定
config/             # requirements.md・HANDOFF.md・tips.md
scripts/            # setup_local_db.py等
tests/
.claude/commands/   # /deploy等のカスタムコマンド
```

---

## 2. 技術スタック

### 標準構成
```
Python 3.12 / FastAPI / uvicorn / Jinja2
HTMX / Alpine.js
boto3 / PyJWT / bcrypt / Mangum
AWS Lambda + DynamoDB
GitHub Actions（CI/CD）
```

### RAG構成（キャリセンアプリ）
```
AWS Bedrock Knowledge Bases（RAGオーケストレーション）
AWS S3 Vectors（ベクトルストア）← OpenSearch Serverlessは高額（約$240/月）
AWS S3（データストレージ）
Titan Text Embeddings V2（埋め込みモデル）
```

---

## 3. DynamoDB設計Tips

### テーブル設計の原則
```python
# PKとSKの命名規則
PK: "USER#user_name"   # プレフィックス#値
SK: "user_id"          # 識別子

# アクセスパターン例
PK = USER#testuser → 特定ユーザーの全データ取得
SK begins_with WORKOUT#202605 → 特定月のデータ取得
```

### よくあるエラー: ValidationException
```
An error occurred (ValidationException) when calling the GetItem operation:
The number of conditions on the keys is invalid
```
**原因**: PKとSKが両方必要なテーブルでPKだけ指定している
```python
# ❌ 間違い
Key={"PK": f"USER#{user_name}"}

# ✅ 正しい
Key={"PK": f"USER#{user_name}", "SK": user_id}
```

### よくあるエラー: ResourceNotFoundException
```
Cannot do operations on a non-existent table
```
**原因①**: テーブル名の大文字小文字が違う（DynamoDBは区別する）
**原因②**: .envのTABLE_NAMEと実際のテーブル名が不一致
**原因③**: DynamoDB Localが停止してテーブルが消えた（-inMemoryオプション使用時）

### DynamoDB Local操作コマンド
```bash
# ※ --no-cli-pager を付けると全件表示される

# テーブル一覧
AWS_ACCESS_KEY_ID=dummy AWS_SECRET_ACCESS_KEY=dummy \
aws dynamodb list-tables \
  --endpoint-url http://localhost:5434 \
  --region ap-northeast-1 --no-cli-pager

# テーブルスキャン
AWS_ACCESS_KEY_ID=dummy AWS_SECRET_ACCESS_KEY=dummy \
aws dynamodb scan \
  --table-name User \
  --endpoint-url http://localhost:5434 \
  --region ap-northeast-1 --no-cli-pager

# データ投入
AWS_ACCESS_KEY_ID=dummy AWS_SECRET_ACCESS_KEY=dummy \
aws dynamodb put-item \
  --table-name User \
  --endpoint-url http://localhost:5434 \
  --region ap-northeast-1 \
  --item '{"PK": {"S": "USER#testuser"}, "SK": {"S": "001"}, ...}'

# テーブル削除
AWS_ACCESS_KEY_ID=dummy AWS_SECRET_ACCESS_KEY=dummy \
aws dynamodb delete-table \
  --table-name User \
  --endpoint-url http://localhost:5434 \
  --region ap-northeast-1
```

**dummyを使う理由**: DynamoDB LocalはAWS認証不要だがCLIが認証情報を要求するためダミー値で回避

### DynamoDB Local起動
```bash
docker run -d -p 5434:8000 amazon/dynamodb-local \
  -jar DynamoDBLocal.jar -inMemory
```
**注意**: -inMemoryオプションはDocker再起動でデータが消える
→ 毎回`python scripts/setup_local_db.py`でデータ再投入が必要

### パスワードハッシュ化
```bash
python -c "import bcrypt; print(bcrypt.hashpw('password'.encode(), bcrypt.gensalt()).decode())"
```

---

## 4. Lambda デプロイTips

### よくあるエラーと解決策

**① ImportModuleError: No module named 'lambda_function'**
```
原因: Lambdaのハンドラー設定がデフォルトのlambda_function.handlerになっている
解決: Lambdaコンソール → コードタブ → ランタイムの設定 → ハンドラー → main.handler
```

**② ImportModuleError: No module named 'pydantic_core._pydantic_core'**
```
原因: MacのARM環境でビルドしたライブラリがLambda（x86_64）で動かない
解決: レイヤーをx86_64でビルドし直す
```
```bash
mkdir -p layer/python
pip install -r requirements.txt \
  -t ./layer/python \
  --platform manylinux2014_x86_64 \
  --only-binary=:all: \
  --python-version 3.12
cd layer && zip -r ../layer.zip . && cd ..
```

**③ ImportModuleError: No module named 'src'**
```
原因: import文にsrc.プレフィックスが残っている
解決: 全ファイルのimport文からsrc.を削除
  src.routers → routers
  src.services → services
  src.repositories → repositories
```

**④ RuntimeError: Directory 'src/static' does not exist**
```
原因: Lambda環境ではsrc/の中身がルートに展開される
解決: パスからsrc/を除去
  StaticFiles(directory="src/static") → StaticFiles(directory="static")
  Jinja2Templates(directory="src/templates") → Jinja2Templates(directory="templates")
```

### deploy.yml（Lambdaレイヤー使用版）
```yaml
name: Deploy to Lambda
on:
  workflow_dispatch:
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: ZIPを作成
        run: |
          cd src
          zip -r ../function.zip .
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1
      - name: Lambdaを更新
        run: |
          aws lambda update-function-code \
            --function-name ${{ secrets.LAMBDA_FUNCTION_NAME }} \
            --zip-file fileb://function.zip
```

### GitHub Secretsに登録するもの
```
AWS_ACCESS_KEY_ID      ← IAMユーザーのアクセスキー
AWS_SECRET_ACCESS_KEY  ← IAMユーザーのシークレットキー
LAMBDA_FUNCTION_NAME   ← Lambda関数名（例: kintai-app）
```

### LambdaのIAMロールとIAMユーザーの違い
```
IAMユーザー（toya-dev）
→ GitHub ActionsがAWSにアクセスするための認証
→ AWSLambdaFullAccess権限が必要

LambdaのIAMロール
→ Lambda自身がDynamoDB等にアクセスするための権限
→ AmazonDynamoDBFullAccess権限が必要
```

### Lambda環境変数（本番）
```
ENV=production
DYNAMODB_REGION=ap-northeast-1
USER_TABLE_NAME=User
JWT_SECRET_KEY=（secrets.token_hex(32)で生成）
JWT_ALGORITHM=HS256
JWT_EXPIRE_HOURS=6
```
```bash
# JWT_SECRET_KEY生成コマンド
python -c "import secrets; print(secrets.token_hex(32))"
```

---

## 5. FastAPI + Jinja2 Tips

### TemplateResponseの正しい書き方（Starlette新バージョン）
```python
# ❌ 古い書き方（TypeError: unhashable type: 'dict'が発生する場合がある）
return templates.TemplateResponse("login.html", {"request": request})

# ✅ 正しい書き方
return templates.TemplateResponse(
    request=request,
    name="login.html",
    context={"key": "value"}
)
```

### HTMX部分更新とサイドバー固定
```python
# HTMXリクエストの判定
is_htmx = request.headers.get("HX-Request")
template = "partials/chat_content.html" if is_htmx else "chat.html"
```
```html
<!-- ナビリンクのHTMX属性 -->
<a hx-get="/chat"
   hx-target="#cn-main-content"
   hx-swap="innerHTML"
   hx-push-url="true">チャット</a>
```

### ナビのアクティブ状態をHTMX更新後に維持する
```html
<!-- partialの末尾にOOBでナビを差し替え -->
<nav id="cn-nav" hx-swap-oob="true" class="cn-nav">
  {% include 'components/nav.html' %}
</nav>
```
→ ナビHTMLをcomponents/nav.htmlに切り出して一元管理する

### .envの読み込み
```python
# CLAUDE.mdに「.envファイルの参照禁止」と書くとload_dotenvが実装されない
# 明示的に許可するか、database.pyに以下を追記してもらう
from dotenv import load_dotenv
load_dotenv()
```

---

## 6. Claude Code活用Tips

### CLAUDE.mdに書くべき禁止事項
```markdown
## 禁止事項
- ハードコードされたダミーデータを本番コードに含めないこと
- TODOコメントを残したまま実装を完了しないこと
- import文にsrc.プレフィックスを含めないこと
- 静的ファイルのパスにsrc/を含めないこと
```

### Claude Designハンドオフの注意点
- ハンドオフはフロントエンド中心の実装を行う
- バックエンドの繋ぎ込みはimplement.mdで別途指示が必要
- ReactベースのHANDOFF.mdはFastAPI+Jinja2向けに変換が必要
  → HANDOFF.md再生成時に「FastAPI + Jinja2 + HTMX + Alpine.js」を明示する

### implement.mdのテンプレート
```markdown
## コンテキスト
- CLAUDE.md
- config/requirements.md
- config/HANDOFF.md

## タスク
### [機能名]
- [具体的な実装指示]

## 注意事項
- [禁止事項・制約]
```

### エラー発生時の対処フロー
```
1. uvicornのログでTraceback最下部のエラーを確認
2. このtips.mdで同じエラーを検索
3. implement.mdに修正指示を記載して/task実行
4. ローカルで動作確認後にプッシュ・デプロイ
```

---

## 7. JWT認証Tips

### JWTのpayloadに含めるべき情報
```python
# create_token関数
payload = {
    "sub": user_id,
    "user_name": user_name,  # ← DynamoDBのPKに使うため必要
    "exp": expire
}
```
**注意**: payloadの構造を変更した場合は既存のCookieを削除して再ログインが必要

### 認証の流れ
```
ログイン → bcryptでパスワード検証 → JWTトークン生成
→ HttpOnly CookieにセットPyJWT
→ 全エンドポイントでDependsによりトークン検証
→ 期限切れ・無効時は/loginにリダイレクト
```

### staff/student権限管理
```python
def require_staff(user=Depends(current_user)):
    if user["role"] != "staff":
        raise HTTPException(
            status_code=307,
            headers={"Location": "/chat"}
        )
    return user
```

---

## 8. ローカル開発とLambda環境の違い

| 項目 | ローカル | Lambda |
|---|---|---|
| import文 | `from src.routers import ...` | `from routers import ...` |
| 静的ファイルパス | `"src/static"` | `"static"` |
| テンプレートパス | `"src/templates"` | `"templates"` |
| DynamoDB | `endpoint_url=http://localhost:5434` | `endpoint_url=None`（本番AWS） |
| 環境変数 | `.env`ファイル | Lambda環境変数 |
| 起動 | `uvicorn src.main:app` | Mangum経由 |

---

## 9. スマートフォンアクセス制限
```python
@app.middleware("http")
async def block_mobile(request: Request, call_next):
    user_agent = request.headers.get("user-agent", "").lower()
    mobile_keywords = ["iphone", "android", "mobile", "blackberry"]
    if any(keyword in user_agent for keyword in mobile_keywords):
        return HTMLResponse(content="...", status_code=403)
    return await call_next(request)
```
**注意**: User-Agentはブラウザ設定で変更できるため完全なブロックではない

---

## 10. RAG・Bedrock Tips

### BedrockはメモリなしのためContextを毎回渡す
```python
# チャット送信時にChatHistoryから過去10件を取得して一緒に送信
messages = [
    {"role": "user", "content": "過去の質問"},
    {"role": "assistant", "content": "過去の回答"},
    ...（最大10件）
    {"role": "user", "content": "今回の質問"}
]
```

### Knowledge Base同期の流れ
```
S3にファイルアップロード
        ↓
start_ingestion_job APIを呼び出す
        ↓
BedrockがS3ファイルを読み込みチャンク分割
        ↓
Titan Text Embeddings V2でベクトル化
        ↓
S3 Vectorsに保存
        ↓
チャットで参照可能になる
```

### コスト比較（ベクトルストア）
```
S3 Vectors: 約$2〜5/月（個人利用）← 推奨
OpenSearch Serverless: 約$240/月 ← 高額のため非推奨
```
