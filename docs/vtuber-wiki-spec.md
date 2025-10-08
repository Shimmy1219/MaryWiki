# Vtuber Wiki 仕様書（暫定 / Markdown）

最終更新: 2025-10-06（JST）

---

## 1. 目的 / コンセプト

* **誰でも編集できる**Vtuber Wiki を **可愛いUI**で提供
* 最初は軽量構成（Option A）で**早期リリース** → 運用安定後に拡張
* 画像・著作権・荒らし対策を事前設計し、**安全運用**を重視

---

## 2. デプロイ / ベース技術

* **ホスティング**: Vercel
* **フロント**: Next.js（App Router, React, TypeScript）
* **スタイリング**: Tailwind CSS + daisyUI（テーマ） + 一部 shadcn/ui
* **エディタ**: TipTap（ProseMirrorベース、WYSIWYG）
* **DB**: PostgreSQL（Vercel Postgres または Neon）+ Prisma
* **ストレージ**: Vercel Blob（画像）
* **認証**: NextAuth.js（Discord OAuth）
* **検索**: 当初はDB検索 → 後日 Meilisearch 導入
* **差分**: jsondiffpatch 等でTipTap JSON差分表示
* **監視**: Sentry（候補）

---

## 3. ルーティング / URL

* `/` … ホーム
* `/search?q=&type=&tag=&sort=` … 検索
* `/wiki/[slug]` … ページ閲覧
* `/edit/[slug]` … ページ編集（要ログイン）
* `/new?type=character|term|stream|clip|timeline` … 新規作成
* `/history/[slug]` … 履歴・差分
* `/tags/[tag]` … タグページ
* `/assets` … 画像ギャラリー
* `/profile` … プロフィール・通知
* `/admin/moderation` … モデレーション（adminのみ）

---

## 4. 役割 / 権限

| 操作        | viewer | editor | admin |
| --------- | ------ | ------ | ----- |
| 閲覧        | ✅      | ✅      | ✅     |
| 新規作成      | ⛔      | ✅（下書き） | ✅     |
| 編集        | ⛔      | ✅（下書き + 即公開可） | ✅     |
| 公開/ロールバック | ⛔      | ✅      | ✅     |
| ユーザー凍結    | ⛔      | ⛔      | ✅     |

* ログインしていない利用者は `viewer`。Discord OAuth でログインし、所定のギルド（サーバー）に参加している場合は自動的に `editor` 権限を付与する。未参加の場合は `viewer` のままとする。
* `editor` は公開ボタンを保持し、自身または他者のドラフトを即時公開できる。
* `admin` はユーザー管理・凍結など運営操作も実施可能。
* すべての編集は**リビジョン履歴**として保存。ワンクリックでロールバック可能。

---

## 5. データモデル（要点）

* `users(id, discord_id, name, avatar_url, role)`
* `pages(id, slug, title, reading, title_image_asset_id, type, created_by, updated_by, is_published)`
* `page_revisions(id, page_id, editor_id, content_json, summary, created_at)`
* `tags(id, name)`／`page_tags(page_id, tag_id)`
* `assets(id, url, uploaded_by, alt, meta)`

> `content_json` は TipTap JSON（構造化ドキュメント）

* `title_image_asset_id` はページタイトル左側に表示する正方形画像を指す。512×512px 以下を推奨し、アップロード時にクライアント側で自動リサイズし、縦横比を 1:1 にクロップする。

### 5.1 TipTap カスタムノード JSON 指針

TipTap のカスタムノードは、`type` にノード名、`attrs` にデータを格納する。`content` があるノードは配下に通常ブロックを保持できる。

#### Infobox (`type: "infobox"`)

```json
{
  "type": "infobox",
  "attrs": {
    "title": "雪見まりあ",
    "thumbnail": { "url": "https://...", "alt": "公式立ち絵" },
    "fields": [
      { "label": "所属", "value": "個人勢" },
      { "label": "初配信", "value": "2024-03-01" },
      { "label": "ファンネーム", "value": "まりあーじゅ" }
    ],
    "links": [
      { "label": "YouTube", "url": "https://youtube.com/@..." },
      { "label": "X", "url": "https://x.com/..." }
    ],
    "colorTheme": "#88E5EF"
  }
}
```

* 必須: `title`
* 任意: `thumbnail`（`url` 必須、`alt` 任意）、`fields[]`（`label`・`value`必須）、`links[]`（`label`・`url`必須）、`colorTheme`
* バリデーション: `fields` は最大20件、文字列は 200 文字以内、URL は https のみ許可

#### Timeline (`type: "timeline"`)

```json
{
  "type": "timeline",
  "attrs": {
    "events": [
      {
        "date": "2023-12-24",
        "title": "デビュー告知",
        "description": "クリスマスイブに初告知" ,
        "media": { "url": "https://...", "type": "image" }
      }
    ]
  }
}
```

* 必須: `events[]`（各要素に `date` or `dateRange` と `title`）
* 任意: `description`、`media`（`type`: `image`/`video`, `url`）
* バリデーション: `events` 最大全体で 100 件、日付は ISO8601 形式、`description` は 500 文字以内

#### Gallery (`type: "gallery"`)

```json
{
  "type": "gallery",
  "attrs": {
    "layout": "grid",
    "images": [
      { "assetId": "asset_123", "url": "https://...", "alt": "1周年記念" }
    ],
    "caption": "公式画像まとめ"
  }
}
```

* 必須: `images[]`（`assetId` or `url` のどちらか必須、`alt` 任意）
* 任意: `layout`（`grid` / `carousel`）、`caption`
* バリデーション: 画像は最大 12 件、`assetId` がある場合はサーバー側で存在チェック

#### YouTube (`type: "youtube"`)

```json
{
  "type": "youtube",
  "attrs": {
    "videoId": "dQw4w9WgXcQ",
    "start": 0,
    "end": null,
    "title": "代表的な配信"
  }
}
```

* 必須: `videoId`
* 任意: `start`（秒数、0 以上の整数）、`end`（`start` より大きい整数 or null）、`title`
* バリデーション: `videoId` は 11 文字の英数字と `_`、`-`

#### Reference (`type: "reference"`)

```json
{
  "type": "reference",
  "attrs": {
    "entries": [
      {
        "label": "公式サイト",
        "url": "https://...",
        "note": "2024-01-01 閲覧"
      }
    ]
  }
}
```

* 必須: `entries[]`（`label`・`url`必須）
* 任意: `note`
* バリデーション: 1 ページ最大 50 件、URL は https のみ

#### Tweet (`type: "tweet"`)

```json
{
  "type": "tweet",
  "attrs": {
    "tweetId": "1234567890123456789",
    "author": "@vtuber",
    "fallbackText": "告知ツイート",
    "lang": "ja"
  }
}
```

* 必須: `tweetId`
* 任意: `author`（表示名 or ハンドル）、`fallbackText`（API 取得失敗時に表示するテキスト）、`lang`
* バリデーション: `tweetId` は 64bit 整数文字列、`fallbackText` は 280 文字以内。
* 表示時は Twitter oEmbed を利用し、ダーク/ライトテーマに合わせてテーマパラメータを変更する。埋め込みスクリプト読み込みが失敗した場合は `fallbackText` をカード表示する。

> サーバー保存時は各ノードの `attrs` を zod で検証し、余分なキーは削除する。既存ノードとの互換性のため、未定義のキーは無視して保存しない。

---

## 6. API（契約概要）

* `GET /api/pages?query=&type=&tag=&limit=&cursor=&sort=` → `{ items, nextCursor }`
* `GET /api/pages/:slug` → `{ page, latestRevision, publishedRevision }`
* `POST /api/pages` → `{ id, slug }`（作成）
* `POST /api/pages/:id/revisions` → `{ revisionId, status }`（要約 `summary` 必須）
* `POST /api/pages/:id/publish`（editor/admin）
* `GET /api/revisions?slug=` → `{ revisions[] }`
* `POST /api/upload` → `{ uploadUrl, assetId }`（署名URL）
* `POST /api/report` → `{ ticketId }`
* 認証フロー: Discord OAuth コールバック時に Bot Token でギルドメンバーシップを検証し、所属していれば `role=editor`、未所属なら `role=viewer` としてセッション/DB を更新する。
* すべて zod バリデーション + Rate limit + CSRF 対策

### 6.1 RevisionStatus 運用ルール

| 状態          | 生成タイミング                                                | 遷移先                                   |
| ------------- | ------------------------------------------------------------- | ---------------------------------------- |
| `DRAFT`       | エディタが `/api/pages/:id/revisions` でリビジョンを作成した時 | `PUBLISHED` / `ROLLED_BACK` |
| `PUBLISHED`   | editor/admin が `/api/pages/:id/publish` を実行した時         | `ROLLED_BACK`（別リビジョンが公開された時）|
| `ROLLED_BACK` | `/api/pages/:id/publish` で過去リビジョンに差し替えた時        | なし（履歴参照のみ）                    |

* 公開時は同一ページ内の既存 `PUBLISHED` リビジョンを `ROLLED_BACK` に更新し、`pages.is_published` を `true`、`pages.updated_by` を公開実行者に更新する。


---

## 7. UIテーマ / デザイントークン

### 7.1 ライトテーマ（`vtLight`）

* **メイン背景**: `#F8F7F5`
* **サブカラー**: `#88E5EF` と `#E2E0B9`
* **4:1 グラデーション**: 左→右 `#88E5EF` 80% → `#E2E0B9` 20%
* **アクセント（金）**: 例 `#C79A00`（装飾/強調）
* **テキスト**: 濃グレー系（例 `#3B3B3B`）
* **角丸**: `rounded-2xl` 以上
* **影**: `shadow-cute`（淡いサブ色影）/ 強調は`shadow-gold`
* **ページ全体の金縁**: 擬似要素 `:before` のリング + ゴールドグロー
* **オーナメント**: SVG（アメジスト、金チェーン）を要所に配置

### 7.2 ダークテーマ（`vtDark`）

* **メイン背景**: `#252525`
* **サブカラー**: `#9B4370` と `#2E3374`
* **4:1 グラデーション（ダーク）**: 左→右 `#9B4370` 80% → `#2E3374` 20%
* **アクセント（金）**: 例 `#E0B84A`
* **テキスト**: 明グレー系（例 `#E8E8E8`）

### 7.3 フォント

* 本文: Noto Sans JP
* 見出し: Zen Maru Gothic（`font-display`）

### 7.4 モーション

* マイクロインタラクション中心（hover ±5% / active -5%）
* `prefers-reduced-motion` 対応

---

## 8. 画面仕様

### 8.1 ホーム（`/`）

* **ヒーロー**: ロゴ、検索バー（≥2文字で候補）、CTA「新規ページ」
* **セクション**

  * 最近の更新（カード一覧：タイトル/更新者/日時/タグ）
  * 注目のページ（ピン留め）
  * タグクラウド（頻度でサイズ変化）
* **レスポンシブ**: md+ は3カラム、sm は1カラム

### 8.2 検索（`/search`）

* **フィルタ**: type（character/term/stream/clip/timeline）、tag、sort（`aiueo` / `created_desc`）
* **ソート定義**:
  * `aiueo` … ページタイトルの読み仮名（`pages.reading` フィールド）を五十音順で昇順ソート。読みが未登録の場合はタイトルをそのままかな変換して使用。
  * `created_desc` … `pages.created_at` の降順。下書き状態のページは含めず、公開済みページのみ対象。
* **結果**: カード表示（スニペット, タグ, 更新時刻）
* **ページング**: 無限スクロール
* **空状態**: 検索Tips

### 8.3 ページ閲覧（`/wiki/[slug]`）

* **ヘッダ**: タイトル、タイトル左の正方形サムネイル（512px 以下）、タグ、編集ボタン（権限）、ブックマーク、通報
* **情報バー**: 最終更新（ユーザー・日時）、閲覧数、リビジョン数
* **本文**: TipTap JSON → SSR安全HTML

  * カスタムノード: Infobox / Timeline / Gallery / YouTube / Tweet / 参考文献
  * Tweet 埋め込みは oEmbed + lazy load。失敗時は `fallbackText` をカード表示。
* **目次（TOC）**: **リッチカードレイアウト**（カテゴリ別折りたたみ + アイコン/サムネ）

  * h2/h3 と `data-category` をもとに生成し、カテゴリごとにガラス質感のカードを横並びレイアウト。各見出しはミニアイコンと読み仮名ラベル付きチップで表示する。
  * 現在位置ハイライト（IntersectionObserver）に加え、固定サイドパネルとしてスクロール追従し、折りたたみアニメーションに 150ms のフェード/スライドを適用。
* **右サイド（md+）**: Infobox（アバター、所属、初配信日、ファンネーム、ファンアートタグ、カラー、公式リンク）
* **フッタ**: 「最終更新の差分を見る」→ `/history/[slug]`、同タグページの横スクロール
* **404/Wanted Page**: 作成CTA + 参考リンク投稿フォーム

### 8.4 ページ編集（`/edit/[slug]`）

* **レイアウト**: 左＝エディタ、右＝サイドパネル（タブ：Infobox / ブロック / 画像 / プレビュー）
* **エディタ機能**:

  * 見出し/太字/斜体/リンク/リスト/表/引用/脚注/コード/画像/ギャラリー/タイムライン/区切り線/スラッシュコマンド
  * 画像: D&D → 一時URL → 保存時 Vercel Blob へ確定
  * タイトルサムネイル: 専用フォームからアップロード/選択。512×512 を上限にクライアントでリサイズし、既存アセットからの選択も可能。
  * オートセーブ（5秒デバウンス） or 手動保存 + 未保存警告
  * 競合: `updatedAt` による楽観ロック
* **サイドパネル（Infobox）**:

  * 項目（例、必須*）

    * 名前*、所属、初配信日（ISO）、身長、誕生日（MM-DD）、ファンネーム、ファンアートタグ、カラー(hex)、公式リンク（YouTube/X/Booth/Fanbox/Site）
  * 入力＝ノード属性へ反映（TipTap `updateAttributes`）
* **保存フロー**:

  * 保存時に**要約（summary）必須**
  * `POST /api/pages/:id/revisions` → 下書き or 即公開（権限による）
  * 成功Toast、履歴リンク表示
* **履歴への導線**: エディタ内ショートカット

### 8.5 履歴・差分（`/history/[slug]`）

* **リスト**: RevID / 編集者 / 要約 / 日時 / サイズ / ステータス
* **比較**: 旧/新リビジョン選択 → 左右比較 or インライン差分

  * カスタムノード属性差分は項目ごとに表示
* **操作**: ロールバック、リンクコピー、通報履歴参照

### 8.6 タグ（`/tags/[tag]`）

* タグ概要・説明（編集可）
* 関連タグ（共起）
* ページ一覧（無限スクロール）

### 8.7 アセット（`/assets`）

* グリッド（サムネ、サイズ、alt、使用ページ）
* フィルタ（拡張子、サイズ、未使用）
* 詳細（EXIF、アップロード者、使用箇所）
* 操作（リネーム、alt編集、差し替え、未使用のみ削除）

### 8.8 プロフィール（`/profile`）

* アバター/名前（Discord同期）
* 自分の編集履歴・下書き一覧
* 通知設定（ウォッチページ更新）

### 8.9 管理 / モデ（`/admin/moderation`）

* 通報キュー（荒らし/著作権/スパム）
* 差分確認→差し戻し/ロールバック/凍結
* Rate limit 状況、IP/UAメトリクス
* 監査ログ

---

## 9. 目次（カテゴリ別折りたたみ）仕様詳細

* **入力**: サーバサイドで本文（TipTap JSON or HTML）を走査し、`[{ id, text, level(2|3), category }]` を生成
* **UI**: `<details>`グループでカテゴリ単位に折りたたみ
* **相互作用**: 現在見出しをハイライト、クリックでスムーススクロール
* **アクセシビリティ**: `aria-label="目次"`

---

## 10. キャラクターページ（特別仕様・全画面）

* **レイアウト**: フルビューポート Hero セクション（85svh 目安）
* **背景**: サブカラーの**4:1グラデーション** + ラジアルグロー
* **装飾**: アメジスト／金チェーンのSVGオーナメント
* **モーション**: Framer Motionによる入場/ホバ—の軽い演出
* **ステージ構成（例）**: Hero → プロフィール（Infobox拡張）→ 年表（Timeline）→ ギャラリー → リンク集
* **テーマ同期**: キャラごとに `primary/sub` を差し替え可能
* **将来**: セクションスナップ、LIVEモード（配信中バッジ）

---

## 11. アクセシビリティ / i18n

* フォーカスリング可視、代替テキスト必須
* 主要テキストのコントラスト 4.5:1 以上
* `prefers-reduced-motion` 対応
* i18n: `ja`デフォルト、`en`準備（固定文言辞書化）

---

## 12. セキュリティ / ポリシー

* 認証: Discord OAuth（NextAuth）
* Rate limit + hCaptcha（投稿/通報）
* XSS対策: 許可ノード限定 + サニタイズ
* 編集ガイドライン/利用規約/プライバシー掲示
* 画像の権利: ガイドライン順守、通報/削除フローあり
* ライセンス（検討）: CC BY-SA 4.0
* 監査ログ: `audit_log(userId, action, target, meta, at)`

---

## 13. パフォーマンス

* ISR（再生成）/ RSC fetch キャッシュ
* Next/Image 最適化・遅延読み込み
* 重要操作にスケルトンUI
* 画像CDN/サムネ自動生成（将来）

---

## 14. ロードマップ（導入順）

1. MVP: 閲覧/検索（簡易）/Discordログイン/編集（下書き）/画像アップ/履歴/公開フロー
2. 可愛いUIテーマ適用（金縁/グラデ/オーナメント/カード類）
3. 目次カテゴリ折りたたみ + 差分ビュー整備
4. キャラページ特別仕様（Hero・ギャラリー）
5. Meilisearch導入、日本語検索強化
6. リアルタイム協調（Yjs/Liveblocks）検討

---

## 15. 検証チェックリスト（抜粋）

* 目次がカテゴリ別に正しく折りたたみ動作する
* ダーク/ライトで4:1グラデーションが正しく切替
* ページ全体の金縁がスクロール/レスポンシブでも破綻しない
* 画像アップロード→保存→再表示でalt保持
* 下書き→承認→公開、差分表示→ロールバックが正常動作

---

## 付録 A: Infobox 項目（案）

* **必須**: 名前
* **任意**: 所属、初配信日（YYYY-MM-DD）、身長、誕生日（MM-DD）、ファンネーム、ファンアートタグ、テーマカラー（hex）、公式リンク（YouTube / X / Booth / Fanbox / Site）

---

## 付録 B: グラデーション定義（CSS例）

```css
/* Light */
.bg-sub-grad {
  background: linear-gradient(90deg, #88E5EF 0%, #88E5EF 80%, #E2E0B9 100%);
}
/* Dark */
.dark .bg-sub-grad-dark {
  background: linear-gradient(90deg, #9B4370 0%, #9B4370 80%, #2E3374 100%);
}
```

---

以上。
