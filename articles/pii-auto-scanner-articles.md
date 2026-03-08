---
title: "AI生成記事のPII漏洩を毎朝自動検出する仕組みを作った"
emoji: "🔍"
type: "tech"
topics: ["python", "security", "automation", "ai", "vps"]
published: true
---

# なぜPIIスキャンが必要になったか

AIで記事を大量生成していると、あるとき気づいた。

生成された記事のドラフトに、**社名がそのまま書いてあった**。

```
僕は[社名]で働くコンサルタントです。
```

プロンプトに「自己紹介を含めて書いて」と指示したら、過去の会話に含まれていた情報をそのまま埋め込んだのだ。AIは記憶しない、と言われているが、**同一セッション内の文脈は拾う**。

それが公開される前に手動でチェックできていたのは運が良かっただけだ。記事が月20本以上になると、手動チェックは破綻する。だから自動化した。

---

## 検出対象にしたPIIの種類

スキャンする情報は6カテゴリ：

| カテゴリ | 例 | 対応 |
|:---|:---|:---|
| APIキー・トークン | `ghp_xxxx`（GitHub PAT）、`AIzaxxxx`（GCP） | 🔴 アラート |
| IPアドレス | VPSの実IPアドレス | 🟡 自動置換 |
| GCPプロジェクトID | `acoustic-skein-xxxxx` | 🟡 自動置換 |
| メールアドレス | 実アドレス | 🟡 自動置換 |
| 会社名 | 本名が推測される社名 | 🟡 自動置換 |
| PEMファイルパス | `key-2026-xx-xx.pem` | 🟡 自動置換 |

**アラート**と**自動置換**で対応を分けたのがポイントだ。

APIキーは「置換すれば安全」ではない。**漏洩した時点でキーを無効化する必要がある**。だから自動修正せずアラートで止める。一方、IPアドレスや会社名は置換で安全になるので自動修正する。

---

## スクリプトの構造

全体は3つの処理からなる。

```python
def main():
    files = collect_files()       # スキャン対象ファイルを収集
    for filepath in files:
        findings = scan_file(filepath)   # PII検出
        fix_file(filepath, findings)     # 自動修正（replaceのみ）
    send_line_notify(token, summary)     # 結果通知
```

### スキャンルールの定義

正規表現とアクションをセットで定義する：

```python
PII_RULES = [
    {
        "name": "GitHub PAT",
        "pattern": r"ghp_[A-Za-z0-9]{36}",
        "action": "alert",
        "exclude": [r"ghp_\[A-Za-z0-9\]"],  # 正規表現説明文は除外
    },
    {
        "name": "VPS IPアドレス",
        "pattern": r"160\.251\.252\.112",
        "action": "replace",
        "replace_with": "[VPS_IP]",
    },
    {
        "name": "会社名",
        "pattern": r"ベイカレントコンサルティング|ベイカレント",
        "action": "replace",
        "replace_with": "大手コンサルファーム",
    },
]
```

`exclude` フィールドが重要だ。記事の中でAPIキーのフォーマットを**正規表現として説明している場合**、それ自体を誤検知してしまう。例外パターンを設定することでノイズを減らす。

### 自動修正処理

```python
def fix_file(filepath, findings):
    replace_findings = [f for f in findings if f["action"] == "replace"]
    with open(filepath) as f:
        content = f.read()
    for finding in replace_findings:
        content = re.sub(finding["pattern"], finding["replace_with"], content)
    with open(filepath, "w") as f:
        f.write(content)
```

シンプルに `re.sub` で置換して上書きする。修正前のバックアップはgitが担う。

---

## VPSにデプロイしてcronで毎朝実行

スクリプトを `/opt/cursor/ai_develop/scripts/pii_scanner.py` に配置し、crontabに追加した：

```bash
# 毎朝7:00 JST（UTC 22:00）に実行
0 22 * * * python3 /opt/cursor/ai_develop/scripts/pii_scanner.py >> /var/log/pii-scanner.log 2>&1
```

スキャン対象ディレクトリ：

```
/opt/zenn-public/articles/     # Zenn公開用リポジトリ
/opt/cursor/ai_develop/content/note/   # note記事
/opt/cursor/ai_develop/content/zenn/   # Zenn記事マスター
```

---

## 実際に検出されたもの

初回実行で1件検出・自動修正された：

```
[AUTO-FIX] 会社名（英語）→ 'a major consulting firm'
  対象: ZENN_ARTICLE_DRAFT.md:11
  変更前: 僕は[社名]で働くコンサルタントです。
```

ドラフトファイルに残っていた記述だ。公開済みファイルではなかったが、公開フローに乗り込む前に検出できた。

---

## 設計で気をつけたこと

**「完全自動修正」にしなかった**。

APIキーが漏洩していた場合、テキストを書き換えても**GitHubのコミット履歴に残る**。正しい対応は「キーを無効化する」「git履歴をBFGで消す」だ。それは人間がやるべき判断なので、アラートで止めて通知するだけにした。

また、**正規表現の誤検知**に注意した。記事の中でAPIキーの形式を説明するコード例が、そのままスキャンに引っかかる。`exclude` パターンを使ってノイズを減らしている。

---

## ログの見方

```bash
# 通常ログ（毎回の実行履歴）
tail -20 /var/log/pii-scanner.log

# アラートログ（検出があった場合のみ記録）
tail -50 /var/log/pii-alerts.log
```

問題なし：
```
2026-03-06 07:00:01 JST スキャン対象: 30 ファイル
2026-03-06 07:00:01 JST ✅ PIIなし。問題ありません。
```

検出あり：
```
2026-03-06 07:00:01 JST [FOUND] article_xx.md: 1件
2026-03-06 07:00:01 JST   🟡 自動修正済 L11: [会社名] [社名]...
```

---

AIを使った情報発信は、出力の品質管理だけでなく**個人情報の管理**も自動化しないとスケールしない。スクリプト全体は150行程度で、標準ライブラリのみで動く。

> noteではこの仕組みを作るに至った実体験（社名を書いてしまった話）を書いています → https://note.com/yoshi_ai_dev

---

*Yoshiki — コンサルタント × AI副業実験中*
