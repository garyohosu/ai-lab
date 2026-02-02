# Sakura Server Deployment: visitor.cgi and db.cgi

## 目的
GitHub リポジトリ garyohosu/cgi の最新版を Sakura レンタルサーバーにデプロイする。

## 対象ファイル
- `api/visitor.cgi` - 訪問者追跡API
- `api/db.cgi` - SQLite データベースAPI
- `api/_data/` - データストレージディレクトリ
- `api/_lib.py` - 共通ライブラリ

## 前提条件
- Sakura サーバーへの SSH アクセスが可能
- SSH 設定: `~/.ssh/config` に Sakura サーバーの設定が記載されている
- サーバー上のディレクトリ: `~/www/cgi/api/`

## デプロイ手順

### 1. Sakura サーバーに SSH 接続
```bash
ssh sakura
```

### 2. CGI ディレクトリに移動
```bash
cd ~/www/cgi/
```

### 3. Git の状態確認
```bash
git status
git remote -v
```

### 4. 最新版を取得（既存のリポジトリがある場合）
```bash
git fetch origin
git pull origin main
```

### 5. リポジトリがない場合は clone
```bash
# 親ディレクトリに移動
cd ~/www/
# 既存の cgi ディレクトリをバックアップ
mv cgi cgi.backup.$(date +%Y%m%d_%H%M%S)
# 新規 clone
git clone https://github.com/garyohosu/cgi.git
cd cgi/api/
```

### 6. ファイルのパーミッション設定
```bash
# CGI ファイルに実行権限を付与
chmod 755 api/*.cgi

# ライブラリは読み取り専用
chmod 644 api/_lib.py

# データディレクトリの作成とパーミッション設定
mkdir -p api/_data/visitors
mkdir -p api/_data/databases
chmod 700 api/_data/visitors
chmod 700 api/_data/databases

# .htaccess の配置確認
ls -la api/_data/databases/.htaccess
```

### 7. データベースディレクトリのセキュリティ確認
```bash
# .htaccess が正しく配置されているか確認
cat api/_data/databases/.htaccess
# 内容: "Deny from all" が記載されていること
```

### 8. CGI の動作確認
```bash
# 現在時刻 API のテスト
curl https://garyo.sakura.ne.jp/cgi/api/now.cgi?tz=jst

# visitor.cgi のテスト（統計取得）
curl https://garyo.sakura.ne.jp/cgi/api/visitor.cgi?action=stats

# db.cgi のテスト（オリジン制限があるため、ブラウザからテスト）
```

### 9. エラーログの確認
```bash
# サーバーのエラーログを確認（パスは環境により異なる）
tail -f ~/www/log/error_log
# または
tail -f ~/log/httpd-error.log
```

## 確認事項チェックリスト

- [ ] `api/visitor.cgi` が存在し、実行権限（755）がある
- [ ] `api/db.cgi` が存在し、実行権限（755）がある
- [ ] `api/_lib.py` が存在する
- [ ] `api/_data/visitors/` ディレクトリが存在し、権限が 700
- [ ] `api/_data/databases/` ディレクトリが存在し、権限が 700
- [ ] `api/_data/databases/.htaccess` が存在し、"Deny from all" が記載されている
- [ ] CORS ヘッダーが正しく設定されている（_lib.py で実装済み）

## トラブルシューティング

### CGI が動かない場合
1. パーミッションを確認: `ls -la api/*.cgi`
2. Shebang を確認: `head -1 api/visitor.cgi` → `#!/usr/local/bin/python3` になっているか
3. エラーログを確認: `tail -50 ~/www/log/error_log`

### データベースが作成されない場合
1. ディレクトリの存在確認: `ls -la api/_data/databases/`
2. 権限を確認: `stat api/_data/databases/`
3. Python から書き込めるか確認:
   ```bash
   python3 -c "import os; os.makedirs('api/_data/databases/test', exist_ok=True); print('OK')"
   ```

### CORS エラーが出る場合
1. `_lib.py` の `print_cors_headers()` を確認
2. ブラウザの DevTools で Response Headers を確認
3. 必要に応じて Apache の設定を確認（通常は不要）

## デプロイ後のテスト

### visitor.cgi のテスト
ブラウザで以下にアクセス:
- https://garyohosu.github.io/ai-lab/visitor-map.html

### db.cgi のテスト
ブラウザで以下にアクセス:
- https://garyohosu.github.io/ai-lab/db-test.html
- "Create Test Table" をクリック
- "Insert Sample Data" をクリック
- "Query All Users" をクリック

## 完了確認
全てのテストが成功したら、デプロイ完了です。

## 備考
- 定期的に `git pull` で最新版に更新することを推奨
- データベースファイル（*.db）は自動的にバックアップすることを推奨
- エラーログは定期的にチェックすること
