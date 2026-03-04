---
title: "Canvaを開かずにアイキャッチを量産する — Gemini + Imagen 3 + Pillow + WordPress REST APIの自動生成パイプライン"
emoji: "🖼️"
type: "tech"
topics: ["python", "gemini", "imagen", "wordpress", "automation"]
published: false
---

# 構成

```
article_generator_html_v2.py
  ↓ 記事タイトル・本文を渡す
eyecatch_generator.py
  ├── Step 1: Gemini API → デザイン設定を決定
  ├── Step 2: Imagen 3 → 背景画像を生成
  ├── Step 3: Pillow → タイトルテキストを合成
  └── Step 4: WordPress REST API → アップロード・アイキャッチ設定
```

# Step 1: Gemini API でデザイン設定を決定

記事の内容からアイキャッチのデザイン方針を決める。サイトごとのブランドカラー・トーンを反映させる。

```python
def generate_eyecatch_design(title: str, content_excerpt: str, 
                               brand_config: dict) -> dict:
    """
    Gemini API に記事情報を渡してデザイン設定を生成する
    """
    prompt = f"""
以下の記事のアイキャッチ画像のデザイン設定を JSON で出力してください。

【記事タイトル】
{title}

【ブランド設定】
- メインカラー: {brand_config['main_color']}
- サブカラー: {brand_config['sub_color']}
- サイトイメージ: {brand_config['image']}

【出力フォーマット】
{{
  "background_prompt": "Imagen 3 への背景画像生成プロンプト（英語・50字以内）",
  "text_color": "#FFFFFF or #333333（背景とのコントラストで選択）",
  "text_position": "center or bottom-left",
  "overlay_opacity": 0.4  // 背景を暗くして文字を読みやすくする度合い（0〜0.7）
}}
"""
    
    response = call_gemini_api(prompt, response_mime_type='application/json')
    return json.loads(response)
```

## ブランド設定の例

```python
# サイト別ブランド設定（4サイト運用中: cbd=健康系, seiketsu=スキンケア, match_lab=マッチングアプリ, dmm=エンタメ）
BRAND_CONFIGS = {
    'seiketsu': {
        'main_color': '#2196F3',
        'sub_color': '#1565C0',
        'image': 'minimalist, clean, scientific, blue tone',
    },
    'cbd': {
        'main_color': '#2E7D32',
        'sub_color': '#66BB6A',
        'image': 'natural, organic, calm, green tone',
    },
    'match_lab': {
        'main_color': '#1A237E',
        'sub_color': '#3949AB',
        'image': 'data-driven, professional, dark blue tone',
    },
}
```

# Step 2: Imagen 3 で背景画像を生成

```python
import vertexai
from vertexai.preview.vision_models import ImageGenerationModel

def generate_background_image(prompt: str, output_path: str) -> str:
    """
    Imagen 3 で背景画像を生成して保存する
    """
    vertexai.init(project='your-project-id', location='us-central1')
    model = ImageGenerationModel.from_pretrained('imagen-3.0-generate-001')
    
    # アイキャッチとして使いやすい横長サイズ
    images = model.generate_images(
        prompt=f"{prompt}, high quality, no text, clean background",
        number_of_images=1,
        aspect_ratio='16:9',
    )
    
    images[0].save(location=output_path)
    return output_path
```

> **ℹ️ 前提**: Imagen 3 の利用には [Vertex AI](https://cloud.google.com/vertex-ai) のセットアップが必要です。GCPプロジェクト作成 → Vertex AI API有効化 → 認証設定の手順を先に完了させてください。

# Step 3: Pillow でタイトルテキストを合成

```python
from PIL import Image, ImageDraw, ImageFont
import textwrap

def compose_eyecatch(background_path: str, title: str, 
                      design: dict, output_path: str) -> str:
    """
    背景画像にタイトルテキストを合成する
    """
    # 背景画像を読み込んで 1200x630 にリサイズ（OGP標準サイズ）
    img = Image.open(background_path).resize((1200, 630))
    
    # オーバーレイ（文字を読みやすくするために背景を暗くする）
    overlay = Image.new('RGBA', img.size, (0, 0, 0, 0))
    draw_overlay = ImageDraw.Draw(overlay)
    opacity = int(255 * design['overlay_opacity'])
    draw_overlay.rectangle([0, 0, 1200, 630], fill=(0, 0, 0, opacity))
    img = Image.alpha_composite(img.convert('RGBA'), overlay)
    
    # テキスト描画
    draw = ImageDraw.Draw(img)
    
    # 日本語フォント（NotoSansJP）
    # VPSでは事前インストールが必要: sudo apt install fonts-noto-cjk
    font_path = '/usr/share/fonts/NotoSansJP-Bold.ttf'
    font_size = 56 if len(title) <= 20 else 44  # タイトルの長さでサイズ調整
    font = ImageFont.truetype(font_path, font_size)
    
    # 自動折り返し（1行20文字を目安に折り返す）
    wrapped_lines = textwrap.wrap(title, width=20)
    
    # テキストを中央または左下に配置
    text_color = design['text_color']
    y_start = 250 if design['text_position'] == 'center' else 450
    
    for i, line in enumerate(wrapped_lines):
        bbox = draw.textbbox((0, 0), line, font=font)
        text_width = bbox[2] - bbox[0]
        x = (1200 - text_width) // 2  # 中央揃え
        y = y_start + i * (font_size + 10)
        
        # テキストシャドウ（読みやすさ向上）
        draw.text((x + 2, y + 2), line, fill='#000000', font=font)
        draw.text((x, y), line, fill=text_color, font=font)
    
    img.convert('RGB').save(output_path, 'JPEG', quality=90)
    return output_path
```

# Step 4: WordPress REST API でアップロード・設定

```python
import requests
from pathlib import Path

def upload_eyecatch_to_wordpress(image_path: str, title: str,
                                  wp_url: str, auth: tuple) -> int:
    """
    WordPress のメディアライブラリに画像をアップロード
    Returns: media_id
    """
    with open(image_path, 'rb') as f:
        filename = Path(image_path).name
        response = requests.post(
            f'{wp_url}/wp-json/wp/v2/media',
            headers={
                'Content-Disposition': f'attachment; filename="{filename}"',
                'Content-Type': 'image/jpeg',
            },
            data=f,
            auth=auth,
        )
    
    media_id = response.json()['id']
    return media_id

def set_featured_image(post_id: int, media_id: int,
                        wp_url: str, auth: tuple) -> None:
    """記事のアイキャッチとして設定"""
    requests.post(
        f'{wp_url}/wp-json/wp/v2/posts/{post_id}',
        json={'featured_media': media_id},
        auth=auth,
    )
```

# パイプライン統合

```python
def generate_and_set_eyecatch(title: str, content: str, post_id: int,
                               service_name: str, wp_config: dict) -> None:
    """
    記事情報を受け取ってアイキャッチ生成・WordPressへの設定まで一括実行
    """
    brand_config = BRAND_CONFIGS[service_name]
    
    # 1. デザイン設定を生成
    design = generate_eyecatch_design(title, content[:500], brand_config)
    
    # 2. 背景画像を生成
    bg_path = f'/tmp/bg_{post_id}.png'
    generate_background_image(design['background_prompt'], bg_path)
    
    # 3. テキスト合成
    final_path = f'/tmp/eyecatch_{post_id}.jpg'
    compose_eyecatch(bg_path, title, design, final_path)
    
    # 4. WordPress にアップロード・設定
    media_id = upload_eyecatch_to_wordpress(
        final_path, title, wp_config['url'], wp_config['auth']
    )
    set_featured_image(post_id, media_id, wp_config['url'], wp_config['auth'])
    
    # 一時ファイル削除
    Path(bg_path).unlink()
    Path(final_path).unlink()
```

# コスト感

| ステップ | 使用API | コスト概算 |
|:---|:---|:---|
| デザイン設定生成 | Gemini 2.0 Flash | 約$0.0001/回 |
| 背景画像生成 | Imagen 3 | 約$0.04/枚 |
| テキスト合成 | Pillow（ローカル） | ¥0 |
| WP アップロード | WordPress REST API | ¥0 |

月50記事生成したとして、画像生成コストは約$2/月。

# まとめ

| ステップ | ツール | 役割 |
|:---|:---|:---|
| デザイン設計 | Gemini API | ブランドに合った設定を自動決定 |
| 背景生成 | Imagen 3 | 高品質な背景画像 |
| テキスト合成 | Pillow | 日本語タイトルの重ね合わせ |
| 設定 | WordPress REST API | アイキャッチとして自動設定 |

パイプライン全体の実装コード・ブランド設定テンプレート・フォント設定は有料パック（note ¥2,980）に収録しています。

---

*Yoshiki — コンサルタント × AI副業実験中 | [@Yoshiki_5medias](https://x.com/Yoshiki_5medias)*
