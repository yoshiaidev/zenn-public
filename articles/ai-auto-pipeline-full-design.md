---
title: "月5,000円・Python・VPS — 4サイト自動運用の全設計図を公開する"
emoji: "🏗️"
type: "tech"
topics: ["python", "gemini", "wordpress", "automation", "vps"]
published: false
---

> **この記事で一言でわかること**: 「4サイト自動運用パイプラインの設計思想とフォルダ構成の全体像」

# はじめに

非エンジニアのコンサルタントが、Gemini API × VPS × WordPress REST API を使って4サイトの記事・X投稿を自動化している。

この記事ではその**全設計図**を公開する。コードの全文は別途有料パックで提供するが、「どんな構造で動いているか」はここで全部見せる。

# 全体パイプライン

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────┐
│ Article_Theme│    │  Gemini API  │    │ Google Sheets│    │ WordPress │
│ (Sheets登録) │───▶│  記事生成    │───▶│  承認フロー  │───▶│ REST API  │
└──────────────┘    └──────────────┘    └──────────────┘    └───────────┘
                                               │
                                               ▼
                                       ┌──────────────┐    ┌───────────┐
                                       │  Gemini API  │    │   X API   │
                                       │  ツイート生成│───▶│  自動投稿 │
                                       └──────────────┘    └───────────┘
```

**設計の核心**: 全工程に「人間の承認ステップ」を1回ずつ挟む。AIの幻覚・法令違反表現を防ぐための意図的な半自動設計。

# フォルダ構成（4サイト完全共通）

```
cbd/
├── .env                                    # APIキー（Git管理外・必須）
├── .spreadsheet.env                        # スプレッドシートID
├── article-generator/
│   ├── post-pages/
│   │   └── article_generator_html_v2.py    # 記事生成メイン（約1,300行）
│   └── utilities/
│       ├── wordpress_publisher.py          # WP REST API投稿（約600行）
│       └── eyecatch_generator.py           # アイキャッチ自動生成
├── social_media/
│   ├── tweet_generator_v3.py              # ツイート生成（ペルソナ別）
│   ├── scheduled_tweet.py                 # 定期実行
│   └── x_poster.py                        # X API投稿
├── scripts/
│   └── generate_tweets_for_spreadsheet.py # 一括ツイート生成
├── google_services/
│   └── google_sheets.py                   # Sheets読み書き
└── config/
    ├── personas.json                       # ペルソナ設定
    └── affiliate_links.json               # アフィリエイト設定
```

`match_lab/`、`seiketsu/`、`dmm/` もまったく同じ構成。`.env` の中身だけ変える。

**この統一構成が最大のレバレッジ**: CBDで動いたコードが即座に他3サイトに展開できる。バグ修正も1回で4サイト直る。

# サービス構成と月額コスト

| サービス | 用途 | 月額 |
|:---|:---|:---|
| ConoHa VPS（1GB RAM） | systemd・cron実行基盤 | ¥1,000程度 |
| Gemini API（2.0 Flash） | 記事・ツイート生成 | 無料枠内 |
| Google Sheets API | 承認フロー・テーマ管理 | 無料 |
| WordPress REST API | 記事投稿・メディアアップロード | ドメイン代のみ |
| X API（Basic） | ツイート自動投稿 | 無料 |
| Cursor | コード生成エディタ | $20/月（約¥3,000） |

**合計: 月4,000〜5,000円で4サイト自動運用。**

# Google Sheetsの設計（全体のハブ）

スプレッドシートが全体のコントロールタワーになっている。

## Article_Theme シート（インプット）

| 列 | 内容 | 例 |
|:---|:---|:---|
| テンプレートID | 記事構成パターン（16種類） | `3`（2商品比較型） |
| テーマ | 記事テーマ文 | 「CBDオイルとCBDグミの違い」 |
| キーワード | SEOキーワード | `CBD,オイル,グミ` |
| ステータス | 生成フロー管理 | `待機` → `生成中` → `生成完了` |

## Article_List シート（アウトプット・承認）

| 列 | 内容 |
|:---|:---|
| タイトル | Gemini生成タイトル |
| 本文HTML | SWELL最適化HTML |
| ステータス | `下書き` → `承認済` → `投稿完了` |
| WP URL | 投稿後のWordPress URL |

**ワークフロー**:
1. `Article_Theme` にテーマを登録（週2回・10分作業）
2. cron が毎朝6時に `article_generator_html_v2.py` を実行
3. 生成記事が `Article_List` に入る
4. 人間がSheetsを確認 → ステータスを `承認済` に変更（1〜3分）
5. cron が `wordpress_publisher.py` を実行 → WordPressに自動投稿

# VPS実行の仕組み

## systemd（常駐・ツイート投稿用）

```ini
# /etc/systemd/system/tweet-cbd.service
[Unit]
Description=CBD Tweet Scheduler
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/cursor/cbd
ExecStart=/usr/bin/python3 social_media/scheduled_tweet.py
Restart=always
RestartSec=300
Environment=PYTHONPATH=/opt/cursor/cbd

[Install]
WantedBy=multi-user.target
```

## cron（定期実行・記事生成用）

```bash
# 記事自動生成（各サイト週1回・6:00 JST = 21:00 UTC前日）
0 21 * * 0 cd /opt/cursor/cbd      && PYTHONPATH=. python3 article-generator/post-pages/article_generator_html_v2.py  # 日曜
0 21 * * 1 cd /opt/cursor/dmm      && PYTHONPATH=. python3 article-generator/post-pages/article_generator_html_v2.py  # 月曜
0 21 * * 2 cd /opt/cursor/match_lab && PYTHONPATH=. python3 article-generator/post-pages/article_generator_html_v2.py  # 火曜
0 21 * * 3 cd /opt/cursor/seiketsu && PYTHONPATH=. python3 article-generator/post-pages/article_generator_html_v2.py  # 水曜
```

# 設計で意図的に「しなかったこと」

**完全自動化をしなかった**。AIが生成するコンテンツには以下のリスクがある。

- 架空の出典（「某○○大学の研究では」）
- 法令違反表現（薬機法・景表法）
- テーマと本文のズレ

これらは承認ステップで人間がキャッチする。1日5分の作業で事故を防ぐトレードオフは合理的だと判断した。

# まとめ

| 設計原則 | 内容 |
|:---|:---|
| 横展開優先 | 4サイト完全共通フォルダ構成 |
| 半自動設計 | 承認ステップで品質を担保 |
| コスト最小 | 月5,000円で4サイト運用 |
| 標準化 | `.env` だけ変えれば別サイトに展開 |

全コード・プロンプトテンプレート・設定ファイルは [有料パック（note）](https://note.com/) で公開予定です。

---

*Yoshiki — コンサルタント × AI副業実験中 | [@Yoshiki_5medias](https://x.com/Yoshiki_5medias)*
