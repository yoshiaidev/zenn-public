---
title: "4アカウント×1日4投稿を自動化した設計図 — ペルソナ・投稿比率・承認フローの全構成"
emoji: "🐦"
type: "tech"
topics: ["python", "x", "twitter", "automation", "gemini"]
published: true
---

# 構成の全体像

```
personas.json（ペルソナ定義）
  ↓
tweet_generator_v3.py（ツイート生成）
  ├── Gemini API でコンテンツタイプ別に生成
  └── Google Sheets の Article_List に書き込み
       ↓
   [人間の承認]
       ↓
   scheduled_tweet.py（定時投稿）
       ↓
   X API（投稿）
```

VPS 上で systemd サービスとして稼働。各サイトごとに独立したプロセス。

# ペルソナ設計

各サイトには `config/personas.json` でペルソナを定義する。

```json
{
  "persona": {
    "name": "CBD専門メディア「CBDの人」",
    "role": "科学的根拠を重視するCBD専門家",
    "tone": "データと体験に基づく信頼性重視のトーン",
    "forbidden_phrases": [
      "ぜひお試しください",
      "いかがでしょうか",
      "〜かもしれません"
    ],
    "content_ratio": {
      "data_analysis": 0.6,
      "lifestyle": 0.3,
      "other": 0.1
    }
  }
}
```

`forbidden_phrases` に「AI生成っぽい言い回し」を明示的に禁止している。これをプロンプトに含めることで、生成されるツイートの質が上がる。

# 投稿比率とコンテンツタイプの実装

```python
import random

CONTENT_TYPES = {
    'data_analysis': 0.6,   # 統計・研究データを紹介するツイート
    'lifestyle': 0.3,        # 生活習慣・体験を絡めたツイート
    'other': 0.1             # その他（告知・問いかけ等）
}

def select_content_type(persona: dict) -> str:
    """ペルソナの投稿比率に基づいてコンテンツタイプを選択"""
    ratios = persona.get('content_ratio', CONTENT_TYPES)
    r = random.random()
    cumulative = 0.0
    
    for content_type, ratio in ratios.items():
        cumulative += ratio
        if r < cumulative:
            return content_type
    
    return 'other'
```

# ツイート生成プロンプト

コンテンツタイプ × ペルソナの組み合わせでプロンプトを動的に組み立てる。

```python
def build_tweet_prompt(persona: dict, content_type: str, article_title: str) -> str:
    base_prompt = f"""
あなたは「{persona['name']}」というX（旧Twitter）アカウントの担当者です。

【キャラクター】
{persona['role']}
トーン: {persona['tone']}

【絶対禁止フレーズ】
{', '.join(persona['forbidden_phrases'])}

【投稿タイプ】
{content_type}

【参考記事】
{article_title}

【ルール】
- 1行目は必ず「数字・問いかけ・常識破壊」のいずれかで始める
- 140字以内
- 敬語禁止（常体で書く）
- 末尾に出典を入れる場合は実在するものだけ

Twitterの投稿文だけを出力してください。
"""
    return base_prompt
```

# 投稿スケジュールの設計

4サイトの投稿時刻は5分ずつずらして衝突を防ぐ。

```json
// config/posting_schedule.json
// サイト名: cbd=健康系, match_lab=マッチングアプリ, seiketsu=スキンケア, dmm=エンタメ
{
  "cbd": {
    "times": ["07:30", "12:15", "19:30", "22:00"]
  },
  "match_lab": {
    "times": ["07:45", "12:30", "19:45", "22:15"]
  },
  "seiketsu": {
    "times": ["07:40", "12:25", "19:40", "22:10"]
  },
  "dmm": {
    "times": ["07:35", "12:20", "19:35"],
    "note": "3投稿のみ（アカウント凍結リスク軽減）"
  }
}
```

# systemd サービス設定

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

`RestartSec=300` で5分ごとに起動チェック。クラッシュしても自動復旧する。

# Google Sheets 承認フロー

生成されたツイートは直接投稿せず、Sheetsの承認キューに入れる。

```python
def queue_tweet_for_approval(sheets, service_name: str, tweet_text: str, 
                              content_type: str) -> None:
    """生成ツイートを承認キューに追加"""
    row = [
        service_name,
        tweet_text,
        content_type,
        datetime.now().isoformat(),
        'pending',   # ← 人間がここを 'approved' に変更するとツイートされる
        '',          # 投稿日時（投稿後に記録）
    ]
    sheets.append_row(row)
```

承認済み（`approved`）になったツイートを `scheduled_tweet.py` が定刻に取得して投稿する。

# 実装して気づいた問題点と対策

## 問題1: 同じようなツイートが連続する

**原因**: 同じ記事からツイートを複数生成すると内容が似る

**対策**: 直近20本のツイートをプロンプトに含めて重複を避けるよう指示

```python
recent_tweets = get_recent_tweets(sheets, limit=20)
prompt += f"\n【重複禁止】以下と似た内容は避けること:\n{recent_tweets}"
```

## 問題2: 投稿タイミングのズレ

**原因**: VPSの時刻とJSTのズレ、スクリプトの処理時間

**対策**: cronの時刻をJST換算でUTCに変換して設定

```bash
# JST 07:30 = UTC 22:30（前日）
30 22 * * * cd /opt/cursor/cbd && PYTHONPATH=. python3 social_media/scheduled_tweet.py
```

## 問題3: X API レートリミット

**原因**: X API Basic プランは月2,400ツイートまで

**対策**: 4アカウント × 月約100投稿 = 月400投稿で余裕あり。ただし急に増やすと制限に引っかかる

# まとめ

| コンポーネント | 役割 | ファイル |
|:---|:---|:---|
| personas.json | ペルソナ・投稿比率の定義 | `config/personas.json` |
| tweet_generator_v3.py | Gemini APIでツイート生成 | `social_media/tweet_generator_v3.py` |
| google_sheets.py | 承認キュー管理 | `google_services/google_sheets.py` |
| scheduled_tweet.py | 定時投稿実行 | `social_media/scheduled_tweet.py` |
| tweet-cbd.service | systemd常駐化 | `/etc/systemd/system/` |

全コード・personas.jsonテンプレート・systemdファイル一式は、シリーズ完結後に有料パックとしてnoteで販売予定です。

---

*Yoshiki — コンサルタント × AI副業実験中 | [@Yoshiki_5medias](https://x.com/Yoshiki_5medias)*
