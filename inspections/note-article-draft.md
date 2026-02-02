# Sakura CGI × GitHub Pages で作る、無料の個人用データベースシステム

## 🎯 この記事で作るもの

**月額数百円のレンタルサーバー + 無料の GitHub Pages** で、本格的なデータベース駆動型ウェブアプリを作ります。

### デモ
- 🗺️ [リアルタイム訪問者マップ](https://garyohosu.github.io/ai-lab/visitor-map.html)
- 🧪 [SQLite データベース API テスト](https://garyohosu.github.io/ai-lab/db-test.html)

---

## 💡 なぜこの構成なのか？

### よくある悩み
- **Firebase / Supabase**: 無料枠はあるが、突然の課金が怖い
- **AWS / GCP**: 高機能だが設定が複雑、初心者には敷居が高い
- **Node.js サーバー**: 常時起動が必要、メンテナンスが大変

### この構成の利点
- ✅ **低コスト**: Sakura レンタルサーバー（月額 131円〜）+ GitHub Pages（無料）
- ✅ **シンプル**: Python CGI だけでバックエンドを構築
- ✅ **安定性**: レンタルサーバーの安定性 + GitHub の可用性
- ✅ **学習コスト**: 複雑なインフラ知識は不要
- ✅ **柔軟性**: SQLite で自由にデータ設計

---

## 🏗️ システム構成

```
[GitHub Pages]  →  HTTPS/CORS  →  [Sakura CGI]  →  [SQLite DB]
   (無料)                          (月額131円〜)      (ファイル)
```

### フロントエンド: GitHub Pages
- 静的 HTML/CSS/JavaScript
- Fetch API で CGI を呼び出し

### バックエンド: Sakura CGI (Python)
- Python 3 で書かれた軽量 CGI スクリプト
- CORS ヘッダーで GitHub Pages からのアクセスを許可
- SQLite で永続化

### データストレージ: SQLite
- ファイルベースの軽量データベース
- `.htaccess` で直接アクセスを禁止

---

## 📦 実装した機能

### 1. 訪問者追跡システム (`visitor.cgi`)

**機能**:
- 訪問者の記録（IP、タイムスタンプ、リファラー）
- 統計情報の取得（総訪問数、今日の訪問数、オンライン数）
- 地図上にピン表示

**セキュリティ**:
- IP アドレスは SHA-256 でハッシュ化
- JSONL 形式で保存（1日1ファイル）

**コード例**:
```python
# visitor.cgi (簡略版)
import json
from datetime import datetime

def record_visit(ip_address):
    hashed_ip = hashlib.sha256(ip_address.encode()).hexdigest()
    record = {
        'visitor_id': hashed_ip[:16],
        'timestamp': datetime.utcnow().isoformat(),
        'location': estimate_location(ip_address)
    }
    with open(f'_data/visitors/{date.today()}.jsonl', 'a') as f:
        f.write(json.dumps(record) + '\n')
```

---

### 2. SQLite 汎用データベース API (`db.cgi`)

**機能**:
- CREATE TABLE: テーブル作成
- INSERT: データ挿入
- SELECT: データ取得（WHERE, ORDER BY, LIMIT 対応）
- UPDATE: データ更新（WHERE 必須）
- DELETE: データ削除（WHERE 必須）
- COUNT: レコード数カウント

**セキュリティ対策**:
- ✅ オリジン制限（`garyohosu.github.io` からのみ）
- ✅ SQL インジェクション対策（プリペアドステートメント）
- ✅ パストラバーサル対策
- ✅ WHERE 句必須（UPDATE/DELETE）

**API 例**:
```javascript
// テーブル作成
const response = await fetch('https://garyo.sakura.ne.jp/cgi/api/db.cgi', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    database: 'testdb',
    action: 'create_table',
    table: 'users',
    schema: {
      id: 'INTEGER PRIMARY KEY AUTOINCREMENT',
      name: 'TEXT NOT NULL',
      email: 'TEXT UNIQUE',
      created_at: 'TEXT DEFAULT CURRENT_TIMESTAMP'
    }
  })
});

// データ挿入
await fetch('https://garyo.sakura.ne.jp/cgi/api/db.cgi', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    database: 'testdb',
    action: 'insert',
    table: 'users',
    data: [
      { name: 'アリス', email: 'alice@example.com' },
      { name: 'ボブ', email: 'bob@example.com' }
    ]
  })
});

// データ取得
const users = await fetch('https://garyo.sakura.ne.jp/cgi/api/db.cgi', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    database: 'testdb',
    action: 'select',
    table: 'users',
    where: { age: { '>=': 20 } },
    order_by: 'created_at DESC',
    limit: 10
  })
});
```

---

## 🛠️ 実装のポイント

### 1. CORS 対応

**問題**: GitHub Pages から Sakura サーバーへのリクエストは CORS エラーになる

**解決**: CGI で CORS ヘッダーを返す

```python
def print_cors_headers():
    print("Content-Type: application/json; charset=utf-8")
    print("Access-Control-Allow-Origin: https://garyohosu.github.io")
    print("Access-Control-Allow-Methods: GET, POST, OPTIONS")
    print("Access-Control-Allow-Headers: Content-Type")
    print()
```

---

### 2. SQLite のセキュリティ

**問題**: データベースファイルが直接ダウンロードされる危険性

**解決**: `.htaccess` で保護

```apache
# _data/databases/.htaccess
Deny from all
```

---

### 3. SQL インジェクション対策

**問題**: ユーザー入力をそのまま SQL に埋め込むと危険

**解決**: プリペアドステートメント

```python
# ❌ 危険な例
query = f"SELECT * FROM users WHERE id = {user_id}"

# ✅ 安全な例
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
```

---

## 📊 動作確認

### テスト 1: テーブル作成
```json
// Request
{
  "database": "testdb",
  "action": "create_table",
  "table": "users",
  "schema": {
    "id": "INTEGER PRIMARY KEY AUTOINCREMENT",
    "name": "TEXT NOT NULL"
  }
}

// Response
{
  "ok": true,
  "result": "table_created"
}
```

### テスト 2: データ挿入
```json
// Request
{
  "database": "testdb",
  "action": "insert",
  "table": "users",
  "data": [
    { "name": "アリス" },
    { "name": "ボブ" }
  ]
}

// Response
{
  "ok": true,
  "count": 2
}
```

### テスト 3: データ取得
```json
// Request
{
  "database": "testdb",
  "action": "select",
  "table": "users"
}

// Response
{
  "ok": true,
  "rows": [
    { "id": 1, "name": "アリス" },
    { "id": 2, "name": "ボブ" }
  ],
  "count": 2
}
```

---

## 🚀 デプロイ手順

### 1. Sakura サーバーへのデプロイ

```bash
# SSH 接続
ssh your-account@your-server.sakura.ne.jp

# CGI ディレクトリに移動
cd ~/www/cgi/

# Git リポジトリをクローン
git clone https://github.com/garyohosu/cgi.git
cd cgi/api/

# パーミッション設定
chmod 755 *.cgi
chmod 644 _lib.py
chmod 700 _data/visitors
chmod 700 _data/databases

# 動作確認
curl https://your-server.sakura.ne.jp/cgi/api/now.cgi
```

### 2. GitHub Pages の設定

```bash
# リポジトリをクローン
git clone https://github.com/garyohosu/ai-lab.git
cd ai-lab/

# HTML ファイルを作成
# (visitor-map.html, db-test.html など)

# GitHub にプッシュ
git add .
git commit -m "Add frontend pages"
git push origin main

# GitHub Pages を有効化
# Settings > Pages > Source: main branch
```

---

## 💰 コスト試算

### Sakura レンタルサーバー
- **ライトプラン**: 月額 131円（年払い）
- **スタンダードプラン**: 月額 437円（年払い）

### GitHub Pages
- **無料**: 容量 1GB、月間転送量 100GB まで

### 合計
- **月額 131円〜** で運用可能！

---

## 🎯 応用例

### 1. TODO アプリ
```javascript
// TODO を追加
await dbApi({
  action: 'insert',
  table: 'todos',
  data: { title: '買い物', done: false }
});

// TODO 一覧を取得
const todos = await dbApi({
  action: 'select',
  table: 'todos',
  where: { done: false }
});
```

### 2. ブログシステム
```javascript
// 記事を投稿
await dbApi({
  action: 'insert',
  table: 'posts',
  data: {
    title: '初投稿',
    content: 'こんにちは！',
    author: 'Alice'
  }
});
```

### 3. アンケートフォーム
```javascript
// 回答を保存
await dbApi({
  action: 'insert',
  table: 'responses',
  data: { question_id: 1, answer: 'はい' }
});
```

---

## ⚠️ 注意点

### 1. パフォーマンス
- SQLite はファイルベースなので、大量の同時アクセスには向かない
- 個人用・小規模プロジェクト向け

### 2. バックアップ
- データベースファイルを定期的にバックアップ
- `rsync` や `scp` でローカルにコピー

### 3. セキュリティ
- オリジン制限を必ず設定
- パスワードが必要な場合は別途認証機構を実装

---

## 🔗 リンク

- **GitHub リポジトリ**: 
  - [garyohosu/cgi](https://github.com/garyohosu/cgi) - バックエンド
  - [garyohosu/ai-lab](https://github.com/garyohosu/ai-lab) - フロントエンド
- **デモ**:
  - [訪問者マップ](https://garyohosu.github.io/ai-lab/visitor-map.html)
  - [データベース API テスト](https://garyohosu.github.io/ai-lab/db-test.html)

---

## 📚 参考資料

- [Sakura レンタルサーバー](https://www.sakura.ne.jp/)
- [GitHub Pages](https://pages.github.com/)
- [Python SQLite3 ドキュメント](https://docs.python.org/3/library/sqlite3.html)

---

## まとめ

**Sakura CGI × GitHub Pages** の組み合わせで、**月額 131円** から本格的なデータベース駆動型ウェブアプリが作れます。

- ✅ 低コスト
- ✅ シンプル
- ✅ 安定性
- ✅ 柔軟性

個人プロジェクトや学習用に最適です。ぜひ試してみてください！

---

**Happy Coding! 🚀**
