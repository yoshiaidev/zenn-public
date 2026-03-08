---
title: "GitHubにAPIキーをpushしてしまった時の完全対応手順"
emoji: "🔑"
type: "tech"
topics: ["security", "git", "github", "python", "dotenv"]
published: true
---

# TL;DR

1. **即座に**対象APIキーを無効化（数分以内）
2. **BFG Repo-Cleaner** でgit履歴から完全削除
3. **新しいキーを発行**して `.env` を更新
4. **再発防止**: `.gitignore` + pre-commit フック

---

# 事故の経緯

`.env.bak` というバックアップファイルを作成し、誤ってgitにコミット・プッシュした。ファイル内には Gemini API キーと X API トークンが含まれていた。

GitHubからの自動検出メール到着まで：**約3分**

# Step 1: 即座にキーを無効化

**ここを最初にやる。git履歴の削除より優先。**

| サービス | 無効化場所 |
|:---|:---|
| Google Cloud (Gemini API) | console.cloud.google.com → APIとサービス → 認証情報 |
| X API | developer.twitter.com → Projects & Apps → キー一覧 |
| OpenAI | platform.openai.com → API keys |
| GitHub Token | github.com → Settings → Developer settings → Personal access tokens |

すべてのキーを無効化してから次のステップへ。

# Step 2: BFG Repo-Cleaner でgit履歴から削除

`git filter-branch` より高速で安全。Javaが必要（`brew install bfg` でも可）。

```bash
# 1. リポジトリのミラークローン（元のリポジトリを汚さないため）
git clone --mirror https://github.com/yourname/yourrepo.git

# 2. BFGで対象ファイルを履歴ごと削除
java -jar bfg.jar --delete-files .env.bak yourrepo.git

# または特定の文字列を置換する場合（secrets.txtにAPIキー文字列を列挙）
java -jar bfg.jar --replace-text secrets.txt yourrepo.git

# 3. 対象リポジトリに移動して履歴を整理
cd yourrepo.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# 4. 強制プッシュ（履歴書き換えのため --force が必要）
git push --force
```

**注意**: `--force` プッシュ後は、チームメンバーが `git pull` ではなく `git clone` でリポジトリを取得し直す必要がある。個人開発なら問題なし。

# Step 3: .gitignore を徹底的に整備

```gitignore
# === 環境変数・シークレット ===
.env
.env.*
*.env
.env.bak
.env.old
.env.backup

# === 認証情報 ===
credentials.json
service-account.json
*.pem
*.p12
*.key

# === Google Drive ショートカット（誤コミット防止）===
*.gsheet
*.gdoc
*.gslides

# === IDE・OS ===
.DS_Store
.vscode/settings.json
```

# Step 4: pre-commit フックで自動検出

コミット前にAPIキーらしき文字列を自動検出する。

```bash
# .git/hooks/pre-commit
#!/bin/sh

# APIキーパターンの検出
PATTERNS=(
    'AIza[0-9A-Za-z\-_]{35}'     # Google API Key
    'sk-[A-Za-z0-9]{48}'          # OpenAI API Key
    'xoxb-[0-9A-Za-z-]+'          # Slack Bot Token
    'ghp_[A-Za-z0-9]{36}'         # GitHub Personal Access Token
    'AAAA[A-Za-z0-9_-]{7}:[A-Za-z0-9_-]{140}'  # Firebase API Key
)

for pattern in "${PATTERNS[@]}"; do
    if git diff --cached | grep -E "$pattern" > /dev/null 2>&1; then
        echo "🚨 ERROR: APIキーらしき文字列が含まれています"
        echo "パターン: $pattern"
        echo "コミットを中断します。.gitignoreを確認してください。"
        exit 1
    fi
done

exit 0
```

```bash
# 実行権限を付与
chmod +x .git/hooks/pre-commit
```

# Step 5: 新しいキーの発行と .env 更新

```bash
# .env.example（実際の値は書かない。変数名のみ）
GEMINI_API_KEY=your_gemini_api_key_here
X_API_KEY=your_x_api_key_here
X_API_SECRET_KEY=your_x_api_secret_here
X_ACCESS_TOKEN=your_access_token_here
X_ACCESS_TOKEN_SECRET=your_access_token_secret_here
```

`.env.example` はGitで管理して、`.env`（実際の値）はGit管理外にする。

# 再発防止チェックリスト

```
□ .gitignore に .env と *.env.* を追加した
□ pre-commit フックを設定した（chmod +x 済み）
□ .env.example を作成してGitで管理している
□ コードへのAPIキーのハードコードがないか確認した
□ 既存のコミット履歴に機密情報が含まれていないか確認した
  （git log -p | grep -E 'AIza|sk-|Bearer ' で確認）
□ チームがいる場合は全員に周知した
```

# まとめ

| フェーズ | 作業 | 優先度 |
|:---|:---|:---|
| 発覚直後 | 対象キーを即座に無効化 | 🔴 最優先 |
| 数時間以内 | BFGで履歴削除 + force push | 🔴 必須 |
| 翌日 | .gitignore 整備 + pre-commit フック | 🟠 必須 |
| 継続 | .env.example での変数名管理 | 🟡 推奨 |

「自分はやらない」ではなく「やっても防げる仕組み」を作ることが個人開発のセキュリティの本質。

> このシリーズでは、4サイト自動運用の全設計を無料で公開中。コード全文は最終回で案内する有料パック（note）に収録予定です。
> noteでは構成・設計の裏側をより詳しく書いています → https://note.com/yoshi_ai_dev

---

*Yoshiki — コンサルタント × AI副業実験中 | [@Yoshiki_5medias](https://x.com/Yoshiki_5medias)*
