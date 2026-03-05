---
title: "Google Sheets × GAS で AI記事の承認パイプラインを作る — ステータス遷移設計から実装まで"
emoji: "📋"
type: "tech"
topics: ["googlesheets", "gas", "python", "automation", "ai"]
published: true
---

# なぜ承認フローが必要か

Gemini API で記事を自動生成して直接 WordPress に投稿していた時期に起きた問題：

- 架空出典（「某○○大学の研究では」）が混入
- 薬機法に触れる断定表現が含まれた
- テーマと本文の内容がズレた記事が投稿された

**人間が30秒見れば気づく問題**を、完全自動化が素通りさせていた。

承認フローは「AI不信」からくる設計ではなく、「AIの出力を最終検査してから公開する」という品質管理の仕組みだ。

# スプレッドシートの設計

## 3シート構成

```
Article_Theme     ← インプット（テーマ登録）
  ↓
Article_List      ← 処理結果・承認管理
  ↓
投稿履歴          ← アーカイブ
```

## Article_Theme シート（インプット）

| 列 | 型 | 内容 | 例 |
|:---|:---|:---|:---|
| A | string | テンプレートID | `3` |
| B | string | テーマ | 「CBDオイルとグミの違い」 |
| C | string | キーワード（カンマ区切り） | `CBD,オイル,グミ` |
| D | string | ステータス | `待機` |
| E | string | カテゴリ | `CBDブランド` |
| F | string | タグ（カンマ区切り） | `CBD,比較` |

## Article_List シート（承認管理）

| 列 | 型 | 内容 |
|:---|:---|:---|
| A | string | サービス名 |
| B | string | タイトル |
| C | text | 本文（HTML） |
| D | string | ステータス |
| E | datetime | 生成日時 |
| F | string | WordPress URL |
| G | string | 警告フラグ |

## ステータス遷移

```
待機 → 生成中 → 生成完了 → 承認済 → 投稿完了
 ↑                              ↓
 └──────── 差し戻し（要修正） ───┘
```

`生成中` は二重実行防止のためのロック状態。スクリプト起動時に `待機` → `生成中` に変え、完了後に `生成完了` にする。

# Python 側の実装

## Sheets の読み書き

```python
import gspread
from google.oauth2.service_account import Credentials

SCOPES = ['https://www.googleapis.com/auth/spreadsheets']

def get_sheets_client():
    creds = Credentials.from_service_account_file(
        'service-account.json',
        scopes=SCOPES
    )
    return gspread.authorize(creds)

def get_pending_themes(spreadsheet_id: str) -> list[dict]:
    """Article_Theme から status=待機 のテーマを取得"""
    client = get_sheets_client()
    sheet = client.open_by_key(spreadsheet_id).worksheet('Article_Theme')
    records = sheet.get_all_records()
    
    return [r for r in records if r['ステータス'] == '待機']

def write_generated_article(spreadsheet_id: str, article: dict) -> None:
    """生成記事を Article_List に書き込む"""
    client = get_sheets_client()
    sheet = client.open_by_key(spreadsheet_id).worksheet('Article_List')
    
    row = [
        article['service'],
        article['title'],
        article['html'],
        '生成完了',
        datetime.now().isoformat(),
        '',           # WP URL（投稿後に記録）
        article.get('warnings', ''),  # 警告フラグ
    ]
    sheet.append_row(row)
```

## 承認済み記事の取得と投稿

```python
def get_approved_articles(spreadsheet_id: str) -> list[dict]:
    """Article_List から status=承認済 の記事を取得"""
    client = get_sheets_client()
    sheet = client.open_by_key(spreadsheet_id).worksheet('Article_List')
    records = sheet.get_all_records()
    
    return [
        {'row_index': i + 2, **r}  # 1-indexed + header行分
        for i, r in enumerate(records) 
        if r['ステータス'] == '承認済'
    ]

def mark_as_published(spreadsheet_id: str, row_index: int, wp_url: str) -> None:
    """投稿完了後にステータスとURLを更新"""
    client = get_sheets_client()
    sheet = client.open_by_key(spreadsheet_id).worksheet('Article_List')
    sheet.update_cell(row_index, 4, '投稿完了')   # ステータス列
    sheet.update_cell(row_index, 6, wp_url)        # WP URL列
```

# GAS で承認ボタンを作る

Sheets のメニューに「承認」ボタンを追加する GAS コード。

```javascript
// Google Apps Script
function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('承認操作')
    .addItem('選択行を承認', 'approveSelectedRows')
    .addItem('全件承認', 'approveAllPending')
    .addToUi();
}

function approveSelectedRows() {
  const sheet = SpreadsheetApp.getActiveSheet();
  const selection = sheet.getActiveRange();
  const startRow = selection.getRow();
  const endRow = startRow + selection.getNumRows() - 1;
  
  let approved = 0;
  for (let row = startRow; row <= endRow; row++) {
    const statusCell = sheet.getRange(row, 4);  // D列: ステータス
    if (statusCell.getValue() === '生成完了') {
      statusCell.setValue('承認済');
      statusCell.setBackground('#d9ead3');  // 緑色でハイライト
      approved++;
    }
  }
  
  SpreadsheetApp.getUi().alert(`${approved}件を承認しました`);
}

function approveAllPending() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet()
    .getSheetByName('Article_List');
  const data = sheet.getDataRange().getValues();
  
  let approved = 0;
  for (let i = 1; i < data.length; i++) {  // ヘッダー行をスキップ
    if (data[i][3] === '生成完了') {  // D列: ステータス（0-indexed）
      sheet.getRange(i + 1, 4).setValue('承認済');
      sheet.getRange(i + 1, 4).setBackground('#d9ead3');
      approved++;
    }
  }
  
  SpreadsheetApp.getUi().alert(`${approved}件を一括承認しました`);
}
```

警告フラグ（架空出典・薬機法違反の疑いがある記事）は G列 に記録し、条件付き書式でオレンジ色にハイライトする。

# 全体フローのまとめ

```
[週2回・10分]
Article_Theme にテーマを10件登録
  ↓
[毎朝6時・自動]
article_generator.py 実行
  → Gemini API で記事生成
  → 品質チェック（薬機法・架空出典）
  → Article_List に書き込み（警告フラグ付き）
  ↓
[毎朝・5〜10分]
人間が Article_List を確認
  → 問題なければ「承認済」に変更（GASボタン）
  → 問題があれば差し戻し or 手修正
  ↓
[自動]
wordpress_publisher.py 実行
  → 承認済記事を WordPress に投稿
  → ステータスを「投稿完了」に更新
```

# まとめ

| コンポーネント | 役割 |
|:---|:---|
| Article_Theme シート | テーマ管理・生成キュー |
| Article_List シート | 生成結果・承認管理 |
| google_sheets.py | Python からの Sheets 操作 |
| GAS スクリプト | ブラウザ上の承認ボタン |
| 警告フラグ | 要確認記事のハイライト |

GASスクリプト・Sheetsテンプレート（コピーして即使える形式）は、シリーズ有料パック（note）に収録予定です。

---

*Yoshiki — コンサルタント × AI副業実験中 | [@Yoshiki_5medias](https://x.com/Yoshiki_5medias)*
