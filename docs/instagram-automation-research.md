# ClaudeでInstagram投稿を自動化する仕組み・ベストプラクティス（2025–2026）

> 調査日: 2026-06-23 ／ 対象リポジトリ: `tsuklio-sg-assets`
> 前提ユースケース: 飲食店が `images/dishes/YYYYMMDD/` 配下で管理する週次の料理写真を、Instagramへ（AI生成キャプション付きで）自動投稿したい。

---

## 0. 結論（先に要点）

- **投稿の「公式かつ唯一安全な道」は Instagram の Content Publishing API（旧 Instagram Graph API / 現 Instagram Platform）一択。** ブラウザ自動操作や非公式ライブラリ（instagrapi 等）は規約違反でアカウント停止リスクがあり、ビジネス用途では避ける。
- **公式APIには前提条件がある**: ①Instagramを**プロアカウント（ビジネス/クリエイター）**にする、②（Facebookログイン方式なら）**Facebookページ**と連携、③Meta App Reviewで権限取得、④画像は**公開HTTPS URL**で配信する必要がある（ローカルファイル直アップロード不可）。
- **Claudeの役割は「投稿そのもの」ではなく「中身の生成」**: 料理写真をClaudeのvision（画像認識）に渡し、**キャプション＋ハッシュタグを日本語で生成**する。投稿APIの呼び出しはスクリプト/ワークフロー側が担当。
- **推奨アーキテクチャ（このリポジトリ向け）**: 画像をGitで管理 → GitHub Pages/CDNで公開URL化 → GitHub Actions（cron）で週次起動 → Claude APIでキャプション生成 → **人間が承認**（PR/ドラフト）→ Instagram Graph APIで投稿。
- **必ず人間の承認ゲートを挟む**: AIキャプションは虚偽情報（存在しない値段・効能など）を生成しうる。完全無人投稿は炎上・誤情報リスクがあるため、`media_publish`の直前に承認ステップを置く。

---

## 1. Instagram公式投稿API（Content Publishing API）

投稿を自動化する正規の手段は Meta の **Instagram Platform**（旧称 Instagram Graph API）。2つの認証方式がある。

| 方式 | Facebookページ | 認証先 | 備考 |
|---|---|---|---|
| **Instagram API with Instagram Login**（2024年7月〜） | **不要** | `graph.instagram.com` | 単一アカウント運用にシンプル |
| **Instagram API with Facebook Login** | **必要** | `graph.facebook.com` | 複数アカウント/Business Manager向け |

### 1.1 アカウント要件・権限
- **Instagramプロアカウント（ビジネス or クリエイター）が必須。** 個人アカウントはAPI投稿不可。
- 権限スコープ（2025年1月27日に旧名称が廃止され刷新）:
  - Instagramログイン: `instagram_business_basic` + `instagram_business_content_publish`
  - Facebookログイン: `instagram_basic` + `instagram_content_publish`（＋ページ系権限）
- **Meta App Review**: 自分が所有しないアカウントへ投稿するには各権限の審査（Advanced Access）が必要。開発モードでは手動追加したテスターのみ動作。

### 1.2 2ステップの投稿フロー
1. **コンテナ作成**: `POST /{ig-user-id}/media` に `image_url`（または `video_url`）と `caption` を渡し、`creation_id` を取得。
2. **公開**: `POST /{ig-user-id}/media_publish` に `creation_id` を渡して投稿確定。
- 動画/リールはコンテナの `status_code` が `FINISHED` になるまでポーリングしてから公開する。画像でも数秒待つのが安全。

### 1.3 画像の要件（重要な落とし穴）
- **画像は公開アクセス可能なURL（`image_url`）でなければならない。** Instagram側がそのURLを取得しに来る。**ローカルファイルの直接アップロードは不可**（バイナリ直アップロードはリール動画のresumable uploadのみ）。
- 形式: **JPEGのみ**（拡張JPEG=MPO/JPS不可）。
- 最大ファイルサイズ: **8MB**。
- アスペクト比: **4:5 〜 1.91:1** の範囲。
- 幅: 最小320px（下回ると拡大）/ 最大1440px（上回ると縮小）。色空間はsRGB。
- 2025年3月24日に画像投稿へ `alt_text` フィールドが追加（リール/ストーリーは対象外）。

### 1.4 レート制限
- **24時間あたりの公開数に上限あり。** 公式は段階的に引き上げてきた経緯があり、情報源により **25 / 50 / 100** と数値がばらつく（古い文書ほど25、最新文書では100の記載）。**カルーセルは1投稿としてカウント。**
- 自分のアカウントの正確な枠は `GET /{ig-user-id}/content_publishing_limit` で確認（`quota_usage` と `config.quota_total`、`quota_duration=86400秒` が返る）。**週次投稿なら上限には到底届かない**ので実害はないが、実装時はこのエンドポイントで実値を確認するのが確実。

### 1.5 カルーセル・リール
- **カルーセル**: 最大10枚（画像/動画混在可）。各子コンテナを `is_carousel_item=true` で作り、親を `media_type=CAROUSEL` + `children`（子IDのカンマ区切り）で作成して公開。
- **リール（動画）**: `media_type=REELS`。MOV/MP4・H.264/HEVC＋AAC・≤1GB・推奨9:16 1080×1920。大きい動画はresumable uploadを使用。
- **ストーリー**: APIでの自動公開は対応が限定的・不安定で、多くの第三者ツールは「リマインダー通知」方式で擬似対応している。確実に自動化したいのは**フィード投稿（画像/カルーセル/リール）**と考えるのが安全。

### 1.6 アクセストークン管理
- 短命トークン（1時間）→ 長命トークン（**60日有効**）に交換。
- 長命トークンは作成24時間後以降に `GET /refresh_access_token` で更新可能、更新ごとに**さらに60日延長**。
- **60日間更新しないと失効**し再認証が必要。→ **トークン自動更新の仕組みは必須**（放置するとある日突然投稿が止まる）。

### 1.7 近年の変更点
- **Instagram Basic Display APIは2024年12月4日に廃止**（読み取り専用で投稿用途ではなかった）。後継が「Instagram API with Instagram Login」。
- 旧スコープ名称は2025年1月27日に廃止 → `instagram_business_*` へ。

---

## 2. Claude APIの組み込み（キャプション・ハッシュタグ生成）

Claudeは**投稿APIを叩く部分ではなく、料理写真から投稿文を生成する部分**を担う。

### 2.1 Vision（画像認識）の基本
- 現行Claudeモデルはすべて画像入力対応（`claude-fable-5` / `claude-opus-4-8` / `claude-sonnet-4-6` / `claude-haiku-4-5` など）。
- 画像の渡し方は3通り: **base64** / **URL** / **Files API**（同じ画像を何度も使うならFiles APIが軽量）。
- 対応形式: JPEG・PNG・GIF・WebP。最大10MB / 8000×8000px。
- 解像度の目安: 料理写真1枚なら **長辺1024〜1568px** で十分（多くのモデルは1568pxを超える解像度を破棄する）。1080×1080の画像で約1,300トークン程度。

### 2.2 Messages API（画像＋テキスト）の構造
```python
import anthropic, base64

client = anthropic.Anthropic()
with open("dish.jpg", "rb") as f:
    img = base64.standard_b64encode(f.read()).decode("utf-8")

resp = client.messages.create(
    model="claude-haiku-4-5",          # 軽量・大量処理向け（§2.4）
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {
                "type": "base64", "media_type": "image/jpeg", "data": img}},
            {"type": "text", "text": "<プロンプト（§2.3）>"},
        ],
    }],
)
```
画像ブロックはテキストブロックの**前**に置く。ブランドのトーンは `system` プロンプトに固定し、Prompt Cachingでコスト削減できる。

### 2.3 プロンプト例（日本語キャプション＋ハッシュタグ）
```
あなたは飲食店のSNS担当者です。この料理写真を分析し、Instagram投稿用の文章を作成してください。
- 料理の見た目・食材・雰囲気を踏まえた、食欲をそそる短い日本語キャプション（絵文字可）
- 関連性の高い日本語/英語のハッシュタグを10〜15個
- 写真から確実に読み取れない情報（価格・原産地・健康効果など）は書かないこと
```
最後の一文（**写真から読み取れない事実を書かせない**）がハルシネーション対策として重要。

### 2.4 モデル選定とコスト（100万トークンあたり 入力/出力）
| モデル | ID | 入力 | 出力 | 用途 |
|---|---|---|---|---|
| Claude Fable 5 | `claude-fable-5` | $10 | $50 | 最高品質のコピー |
| Claude Opus 4.8 | `claude-opus-4-8` | $5 | $25 | 高品質 |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | $3 | $15 | 品質/コストの好バランス |
| Claude Haiku 4.5 | `claude-haiku-4-5` | $1 | $5 | **大量・低コスト（推奨）** |

- **推奨: `claude-haiku-4-5`。** 写真1枚＋短いプロンプト＋約300トークン出力で**1枚あたり1円未満**。キャプションの創造性を重視するなら `claude-sonnet-4-6`。
- 大量バッチなら **Batches API（50%割引）** ＋ system promptの **Prompt Caching** でさらに削減。

### 2.5 構造化出力（caption + hashtags をJSONで受け取る）
`output_config.format` にJSON Schemaを渡すと、パース可能なJSONを保証できる（後段の投稿処理に流しやすい）:
```python
output_config={"format": {"type": "json_schema", "schema": {
    "type": "object",
    "properties": {
        "caption": {"type": "string"},
        "hashtags": {"type": "array", "items": {"type": "string"}},
    },
    "required": ["caption", "hashtags"],
    "additionalProperties": False,
}}}
```
あるいはstrict tool useで `save_instagram_post(caption, hashtags)` のような関数を定義してもよい。

---

## 3. アーキテクチャの選択肢

### 3.1 ノーコード/ローコード（n8n / Make / Zapier）
いずれも内部的には**公式Instagram Graph APIを叩く**ため、§1の前提（ビジネスアカウント＋ページ＋公開画像URL）は共通。

| 観点 | Zapier | Make.com | n8n |
|---|---|---|---|
| 手軽さ | 最も簡単・UI洗練 | 簡単・ビジュアル | やや高度（HTTP/コミュニティノード） |
| IG画像投稿アクション | あり（Publish Photo） | あり（Create a Photo Post） | **ネイティブなし**（Facebook Graph/HTTP/コミュニティノード） |
| 課金 | タスク単位（各ステップ課金で割高） | オペレーション単位（最安級 〜$9–11/月） | 実行単位（クラウド）or **セルフホストで無料** |
| 柔軟性 | 低 | 中 | **最高**（フルHTTP/コード） |
| AIキャプション | ネイティブ手順 | ネイティブ手順 | **Anthropicノード**（Opus/Sonnet/Haiku）＋HTTP |

- **非技術者の小規模店なら Make.com** が「専用の写真投稿アクション＋安価＋AI連携容易」でバランス良。
- **セルフホストで最安・最柔軟なら n8n**（無料CE、ネイティブClaudeノードあり）。ただしIG専用ノードがなくGraph API手組みが必要。
- Zapierは最も簡単だが割高で、2024年の旧API廃止で移行トラブルの報告も多い。

### 3.2 Claude Code / Agent SDK / MCP
- **Instagram用のMCPサーバーは複数存在**（すべて第三者製。公式Anthropic製はなし）。例: `mikusnuz/meta-mcp`（Graph API v25、`ig_publish_photo` 等33ツール）、`oliverames/meta-mcp-server`（200+ツール）、`mcpware/instagram-mcp`、`jlbadano/ig-mcp` 等。**トークンの扱い・保守状況を必ず吟味**してから投稿権限トークンを預けること。
- **Claude Code headless（`claude -p`）/ Agent SDK** をCIで実行可能。`--output-format json` ＋ `--json-schema` で機械可読出力、`--allowedTools` / `--permission-mode` で承認待ちを回避、`--mcp-config` でMCPサーバーを読み込み。
- **GitHub Actions公式 `anthropics/claude-code-action`** はcronトリガー対応で、定期実行の自動化に使える。認証は `ANTHROPIC_API_KEY` をSecretに。

### 3.3 自前スクリプト＋cron / GitHub Actions（このリポジトリに最適）
画像をGit管理しているこのリポジトリの構成にそのまま乗る:
1. **画像の公開URL化**: GitHub Pages（`raw.githubusercontent.com` でも可）/ S3 / CDN で `images/dishes/YYYYMMDD/*.jpg` を公開HTTPSに。← Graph APIの必須要件。
2. **GitHub Actions（cron）で週次起動**: 投稿対象の週フォルダ/未投稿画像を選択（日付 or キュー用JSON or "次の未投稿"ポインタ）。
3. **Claude APIでキャプション＋ハッシュタグ生成**（§2）。
4. **人間の承認ゲート**（§4）。
5. **Instagram Graph APIで2ステップ投稿**（§1.2）。`curl`/HTTP直叩き、またはMCPサーバー経由。

---

## 4. ベストプラクティス・注意点（2025–2026）

### 4.1 公式API以外は使わない（2025年のBAN波に注意）
- ブラウザ自動操作（Selenium/Playwright）・instagrapi等の非公式プライベートAPIは **Instagram利用規約違反**で、シャドウバン/アカウント停止のリスク。ビジネスアカウントで使うべきではない。
- **2025年5〜8月頃に大規模な「BAN波」が発生**し、非準拠の第三者アプリ連携に起因する凍結が約30%増加、Metaは上半期に約1,000万件の自動化関連アカウントを削除したと報じられた。AIモデレーションの強化と自動化ポリシーの厳格運用が背景。
- 非公式ツールは**技術的にも壊れやすい**: TLSフィンガープリント遮断、内部GraphQLパラメータの2〜4週ごとのローテーション、不自然な投稿タイミングを検知する挙動MLなど。instagrapi自身も「アカウント所有者のワークフローには公式APIを推奨」している。
- **シャドウバン**は通知なしにリーチ/ハッシュタグ/発見タブ露出が静かに低下する。主なトリガーは大量フォロー/自動いいね/自動コメント、自動ログイン、同一キャプション/ハッシュタグの反復、スパム的な多用ハッシュタグ（#love #happy 等）。
- スケジューラを使うなら **Meta公式パートナー型**（Later, Buffer, Hootsuite, Metricool, Meta Business Suite Planner）を選ぶ。これらは内部で**公式Content Publishing API**を使うため準拠（要プロアカウント）。Meta Business Suite Plannerは無料・純正。

### 4.2 人間の承認ゲートを必ず挟む（最重要）
- AIキャプションは**事実の捏造（存在しない価格・効能・原材料など）**を起こしうる。ブランドはAI生成物にも責任を負う。
- 緩和策: cron投稿を即時公開にせず、**生成キャプション＋画像をPR / Slack / ドラフトコンテナで提示 → 人が承認 → `media_publish`**。MCPサーバーの「ドラフトコンテナで止める」機能や、コンテナ作成までを自動・公開だけ手動にする運用が安全。
- 生成キャプションは**全件ログ保存**して監査証跡を残す。

### 4.3 認証情報の管理
- 長命トークンは**60日で失効**（更新は「作成24時間後〜失効前」のみ可）。失効すると復旧不能なので、**約50〜55日ごとに自動更新するジョブ**を用意する。
- トークンはGitHub Secrets等の**暗号化ストレージ**に保存（リポジトリにコミットしない／平文・クライアント側コード・URLに露出させない）。`app secret` と長命トークンの取得は**必ずサーバー側のみ**で行う。`ANTHROPIC_API_KEY` も同様にSecret管理。
- 漏洩した長命トークンは最大60日間フルアクセスを許してしまうため、アクセスログと異常検知も併用。

### 4.4 投稿頻度・コンテンツ
- Graph API全体の呼び出し枠は概ね**200コール/時/ユーザー**、公開投稿は24時間あたり上限あり（§1.4）。週次の料理写真程度ならいずれにも全く届かない。`content_publishing_limit` で枠を監視しておけば安心。
- フラグ回避の基本: 上限に余裕を持つ／投稿を一気に固めず分散させる／キャプション・ハッシュタグを毎回少し変える／スパム的多用ハッシュタグを避ける。
- `alt_text`（代替テキスト）を付けるとアクセシビリティとSEOに有利。Claudeに画像説明も同時生成させると効率的。

---

## 5. このリポジトリ向けの推奨構成（まとめ）

```
[Git: images/dishes/YYYYMMDD/*.jpg]
        │  (GitHub Pages / CDN で公開HTTPS URL化)
        ▼
[GitHub Actions cron（毎週）]
        │  対象週の画像を選択
        ▼
[Claude API (claude-haiku-4-5, vision)]
        │  caption + hashtags + alt_text を JSON で生成
        ▼
[承認ゲート: PR or Slack 通知 で人が確認]   ← 必須
        ▼
[Instagram Graph API]
   1. POST /media        (image_url, caption)
   2. POST /media_publish (creation_id)
        │
   別ジョブ: refresh_access_token で60日ごとにトークン更新
```

- **最小構成**: GitHub Actions ＋ `curl` ＋ Claude API（追加SaaS不要、無料枠内）。
- **非技術者運用**: Make.com（写真投稿アクション＋AIステップ）。
- **柔軟性重視**: n8n セルフホスト（無料、ネイティブClaudeノード）。

---

## 出典

**Instagram Content Publishing API**
- https://developers.facebook.com/docs/instagram-platform/content-publishing/
- https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/
- https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/media/
- https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/media_publish/
- https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/content_publishing_limit/
- https://developers.facebook.com/docs/instagram-platform/reference/refresh_access_token/
- https://developers.facebook.com/docs/instagram-platform/overview/
- https://elfsight.com/blog/instagram-graph-api-complete-developer-guide-for-2026/
- https://postproxy.dev/blog/post-to-instagram-via-api/

**Claude API（vision・モデル・構造化出力）**
- https://platform.claude.com/docs/en/build-with-claude/vision.md
- https://platform.claude.com/docs/en/about-claude/models/overview.md
- https://platform.claude.com/docs/en/pricing.md
- https://platform.claude.com/docs/en/build-with-claude/structured-outputs.md
- https://platform.claude.com/docs/en/build-with-claude/batch-processing.md

**ノーコード（n8n / Make / Zapier）**
- https://n8n.io/workflows/4498-schedule-and-publish-all-instagram-content-types-with-facebook-graph-api/
- https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatanthropic/
- https://apps.make.com/instagram-business
- https://help.zapier.com/hc/en-us/articles/32429170578317-Instagram-app-deprecation-on-Dec-4-2024
- https://www.sitepoint.com/how-i-automated-multi-platform-social-posting-with-claude-and-n8n-and-stopped-logging-into-5-dashboards-every-morning/

**自動化のリスク・規約・準拠スケジューラ**
- https://help.instagram.com/termsofuse
- https://www.facebook.com/legal/automated_data_collection_terms
- https://blog.postly.ai/why-instagram-is-cracking-down-on-third-party-apps-in-2025/
- https://fanbaseaccelerator.com/blog/instagram-2025-ban-wave
- https://contentstudio.io/blog/instagram-shadowban
- https://creatorflow.so/blog/is-instagram-automation-safe-2026/
- https://blog.hootsuite.com/how-to-schedule-instagram-posts/
- https://github.com/subzeroid/instagrapi
- https://feedframer.com/guides/instagram-long-lived-access-token

**Claude Code / Agent SDK / MCP**
- https://code.claude.com/docs/en/headless
- https://code.claude.com/docs/en/github-actions
- https://github.com/anthropics/claude-code-action
- https://github.com/mikusnuz/meta-mcp
- https://github.com/oliverames/meta-mcp-server
- https://github.com/mcpware/instagram-mcp
- https://www.mirra.my/en/blog/mcp-instagram-content-automation-guide-2026
