---
title: "AIへの指示を16パターンに分類したら記事がブレなくなった — テンプレートシステムの設計と実装"
emoji: "📝"
type: "tech"
topics: ["python", "gemini", "llm", "prompt", "automation"]
published: false
---

> **この記事で一言でわかること**: 「プロンプトに構成テンプレートを組み込むとLLMの出力の構造的ブレがなくなる」

# 問題

Gemini API で記事を量産していると、テーマが違うのに全記事が同じ構成になる現象が起きた。

```
# どのテーマでも同じ構成になっていた（問題のある状態）
H2: ○○とは何か
H2: ○○のメリット
H2: ○○のデメリット
H2: ○○の選び方
H2: まとめ
```

LLM は「汎用的な解説記事」の構成として最もよく学習しているこのパターンを選びやすい。

# 解決策：記事テンプレートシステム

記事の「目的」に応じた構成テンプレートを事前定義し、プロンプトに組み込む。

## テンプレート定義

```python
TEMPLATES = {
    # 【商品・購入系】
    1: {
        "name": "単一商品レビュー型",
        "require_product": True,
        "category": "review",
        "structure": """
H2: {商品名}を選んだ理由（導入）
H2: {商品名}の基本情報（成分・価格・購入先）
H2: 実際に使ってみた（体験・感想）
H2: メリット・デメリット
H2: こんな人におすすめ / こんな人には向かない
H2: よくある質問
H2: まとめ・総合評価
"""
    },
    2: {
        "name": "TOP10ランキング型",
        "require_product": True,
        "category": "ranking",
        "structure": """
H2: 選定基準（何を見て選んだか）
H2: TOP10ランキング一覧（表形式）
H2: 1位〜3位の詳細解説
H2: 予算別おすすめ
H2: 選び方のポイント
H2: よくある質問
H2: まとめ
"""
    },
    3: {
        "name": "2商品比較型",
        "require_product": True,
        "category": "comparison",
        "structure": """
H2: {商品A}と{商品B}の違いを一言で
H2: {商品A}の特徴・メリット・デメリット
H2: {商品B}の特徴・メリット・デメリット
H2: 比較表（スペック・価格・特徴）
H2: どちらを選ぶべきか（目的別の結論）
H2: よくある質問
H2: まとめ
"""
    },
    # 【知識・解説系】
    8: {
        "name": "Q＆A型",
        "require_product": False,
        "category": "faq",
        "structure": """
H2: この記事でわかること
H2: よくある質問10選
  （Q: ... A: ...）の形式で10問
H2: より詳しく知りたい人へ
H2: まとめ
"""
    },
    9: {
        "name": "完全ガイド型",
        "require_product": False,
        "category": "guide",
        "structure": """
H2: この記事でわかること（箇条書き3〜5個）
H2: ○○の基礎知識
H2: 具体的な方法・手順（ステップバイステップ）
H2: よくある間違い・注意点
H2: よくある質問（Q&A 3〜5個）
H2: まとめと次のアクション
"""
    },
    10: {
        "name": "法律・規制解説型",
        "require_product": False,
        "category": "legal",
        "structure": """
H2: 結論（法律的にどうなのか）
H2: 関連法律の概要（わかりやすく）
H2: 具体的に何がOKで何がNGか
H2: 違反した場合のリスク
H2: 安全に利用するための注意点
H2: よくある質問
H2: まとめ
"""
    },
    # 【ショート解説型】
    16: {
        "name": "ショート解説型",
        "require_product": False,
        "category": "short",
        "structure": """
H2: 一言まとめ（結論から書く）
H2: 詳しい解説（300〜500字）
H2: 関連情報・参考リンク
"""
    },
}
```

## プロンプトへの組み込み

```python
def build_article_prompt(theme: str, template_id: int, keywords: list[str],
                          persona: dict, word_count: int = 3000) -> str:
    
    template = TEMPLATES[template_id]
    structure = template['structure']
    
    prompt = f"""
あなたは「{persona['name']}」というWebメディアの専門ライターです。

【記事テーマ】
{theme}

【キーワード】
{', '.join(keywords)}

【記事構成（必ずこの構成に従ってください）】
{structure}

【文字数】
{word_count}字程度（H2の数に応じて均等に配分）

【品質ルール】
- 出典は実在する研究・調査のみ引用（架空の大学名・研究は絶対禁止）
- 断定的な医療効果の表現は禁止
- 「まず〜とは何でしょうか」という冒頭は禁止
- 各H2は300字以上

【出力形式】
HTMLで出力してください。H1タグは使用しないこと。
"""
    return prompt
```

# require_product フラグの活用

```python
def generate_article(theme_row: dict, persona: dict) -> dict:
    template_id = int(theme_row['テンプレートID'])
    template = TEMPLATES[template_id]
    
    # アフィリエイト記事かどうかで処理を分岐
    if template['require_product']:
        # 商品情報をプロンプトに追加
        affiliate_info = load_affiliate_links(theme_row['カテゴリ'])
        prompt = build_article_prompt(
            theme=theme_row['テーマ'],
            template_id=template_id,
            keywords=theme_row['キーワード'].split(','),
            persona=persona,
            affiliate_context=affiliate_info  # ← 商品情報を追加
        )
    else:
        prompt = build_article_prompt(
            theme=theme_row['テーマ'],
            template_id=template_id,
            keywords=theme_row['キーワード'].split(','),
            persona=persona,
        )
    
    html = call_gemini_api(prompt)
    return {'html': html, 'template_name': template['name']}
```

# テンプレート効果の計測

テンプレートシステム導入前後で、承認フローでの「差し戻し率」が変化した。

| 期間 | 生成記事数 | 差し戻し数 | 差し戻し率 |
|:---|:---|:---|:---|
| 導入前（テンプレートなし） | 30本 | 11本 | 37% |
| 導入後（16テンプレート） | 50本 | 6本 | 12% |

構成の指定だけで差し戻し率が25ポイント下がった。

# まとめ

| 設計ポイント | 内容 |
|:---|:---|
| テンプレートの分類 | 目的別（商品系・知識系・課題解決系・ショート型）に16パターン |
| 構成の固定 | H2見出し構成をプロンプトに組み込む |
| require_product フラグ | アフィリエイト記事の自動判別 |
| 差し戻し率の低下 | 37% → 12%（承認フロー計測） |

全16パターンの構成定義・プロンプトテンプレート・実装コードは有料パック（note ¥2,980）で公開予定です。

---

*Yoshiki — コンサルタント × AI副業実験中 | [@Yoshiki_5medias](https://x.com/Yoshiki_5medias)*
