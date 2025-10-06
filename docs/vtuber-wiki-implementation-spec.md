# Vtuber Wiki 実装仕様書

## 1. プロジェクト構成

### 1.1 リポジトリルート

```
/
├── README.md
├── docs/
│   ├── vtuber-wiki-spec.md                 # 既存のプロダクト仕様
│   └── vtuber-wiki-implementation-spec.md  # 本ドキュメント（実装仕様）
├── prisma/
│   ├── schema.prisma                       # Prisma モデル定義
│   └── migrations/                         # マイグレーションファイル群
├── src/
│   ├── app/                                # Next.js App Router ページ/レイアウト
│   │   ├── api/                            # Route Handlers（REST API）
│   │   │   ├── pages/                      # /api/pages
│   │   │   ├── revisions/                  # /api/revisions
│   │   │   ├── upload/                     # /api/upload
│   │   │   └── report/                     # /api/report
│   │   ├── (marketing)/                    # LP・静的ページ
│   │   ├── layout.tsx                      # ルートレイアウト（テーマ適用）
│   │   └── page.tsx                        # ホーム
│   ├── components/                         # 再利用 UI コンポーネント
│   ├── features/                           # ドメイン単位の UI / ロジック
│   ├── hooks/                              # 共通 React Hooks
│   ├── lib/                                # サーバ/クライアント共通ライブラリ
│   ├── server/                             # サーバサイド専用ロジック
│   ├── styles/                             # Tailwind, DaisyUI, カスタムCSS
│   └── types/                              # 型定義
├── public/                                 # 画像・フォントなどの静的アセット
├── scripts/                                # デプロイ/Lint 等の補助スクリプト
└── package.json
```

### 1.2 採用技術

- Next.js 13+（App Router, React Server Components, ISR）
- TypeScript
- Tailwind CSS + daisyUI + shadcn/ui
- TipTap（ProseMirror ベース WYSIWYG）
- Prisma + PostgreSQL（Vercel Postgres / Neon）
- Vercel Blob（画像ストレージ）
- NextAuth.js（Discord OAuth 認証）
- jsondiffpatch（差分表示）
- Sentry（監視候補）

## 2. データレイヤ

### 2.1 Prisma スキーマ（ベース案）

```
model User {
  id          String   @id @default(cuid())
  discordId   String   @unique
  name        String
  avatarUrl   String?
  role        Role     @default(EDITOR_PENDING)
  createdPages Page[]  @relation("PageCreator")
  updatedPages Page[]  @relation("PageUpdater")
  revisions   PageRevision[]
  assets      Asset[]
  auditLogs   AuditLog[]
  reports     Report[] @relation("Reporter")
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

enum Role {
  VIEWER
  EDITOR_PENDING
  CURATOR
  ADMIN
}

model Page {
  id            String         @id @default(cuid())
  slug          String         @unique
  title         String
  type          PageType
  contentType   ContentType    @default(TIPTAP_JSON)
  createdBy     String
  creator       User           @relation("PageCreator", fields: [createdBy], references: [id])
  updatedBy     String
  updater       User           @relation("PageUpdater", fields: [updatedBy], references: [id])
  isPublished   Boolean        @default(false)
  publishedAt   DateTime?
  createdAt     DateTime       @default(now())
  updatedAt     DateTime       @updatedAt
  tags          PageTag[]
  revisions     PageRevision[]
  wantedBy      WantedPage?
}

enum PageType {
  CHARACTER
  TERM
  STREAM
  CLIP
  TIMELINE
}

enum ContentType {
  TIPTAP_JSON
}

model PageRevision {
  id         String          @id @default(cuid())
  pageId     String
  page       Page            @relation(fields: [pageId], references: [id])
  editorId   String
  editor     User            @relation(fields: [editorId], references: [id])
  content    Json
  summary    String
  status     RevisionStatus  @default(DRAFT)
  createdAt  DateTime        @default(now())
}

enum RevisionStatus {
  DRAFT
  PENDING_REVIEW
  PUBLISHED
  ROLLED_BACK
}

model Tag {
  id        String    @id @default(cuid())
  name      String    @unique
  pages     PageTag[]
  description String? @db.Text
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model PageTag {
  pageId String
  tagId  String
  page   Page @relation(fields: [pageId], references: [id])
  tag    Tag  @relation(fields: [tagId], references: [id])
  @@id([pageId, tagId])
}

model Asset {
  id          String   @id @default(cuid())
  url         String
  uploadedBy  String
  uploader    User     @relation(fields: [uploadedBy], references: [id])
  alt         String?
  metadata    Json?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  usages      AssetUsage[]
}

model AssetUsage {
  id        String   @id @default(cuid())
  assetId   String
  pageId    String?
  revisionId String?
  asset     Asset   @relation(fields: [assetId], references: [id])
  page      Page?   @relation(fields: [pageId], references: [id])
  revision  PageRevision? @relation(fields: [revisionId], references: [id])
  meta      Json?
}

model Report {
  id          String   @id @default(cuid())
  reporterId  String
  reporter    User     @relation("Reporter", fields: [reporterId], references: [id])
  pageId      String?
  revisionId  String?
  reason      String
  status      ReportStatus @default(OPEN)
  createdAt   DateTime @default(now())
  resolvedAt  DateTime?
}

enum ReportStatus {
  OPEN
  UNDER_REVIEW
  RESOLVED
  DISMISSED
}

model AuditLog {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  action    String
  target    String
  meta      Json?
  createdAt DateTime @default(now())
}

model WantedPage {
  id        String   @id @default(cuid())
  slug      String   @unique
  requests  Int      @default(0)
  page      Page?    @relation(fields: [slug], references: [slug])
}
```

### 2.2 補助テーブル

- `watch_list(pageId, userId, createdAt)`：ウォッチ通知設定
- `notifications(id, userId, type, payload, readAt)`：通知配信キュー
- `bookmark(pageId, userId, createdAt)`：お気に入り（ブックマーク）

## 3. API 設計（Route Handlers）

| Method | Path | 概要 | 主な処理 | 認可 |
|--------|------|------|----------|------|
| GET | /api/pages | ページ一覧・検索 | クエリ（`query`,`type`,`tag`,`sort`,`cursor`,`limit`）でページネーション検索 | Public |
| GET | /api/pages/[slug] | ページ詳細 | 公開版＋最新下書きを取得。閲覧数計測は別途 Edge Function | Public |
| POST | /api/pages | 新規作成 | slug 生成、下書きページ作成、監査ログ記録 | Editor+ |
| POST | /api/pages/[id]/revisions | リビジョン保存 | TipTap JSON と `summary` を保存。`role` に応じて `status` 切替 | Editor+ |
| POST | /api/pages/[id]/publish | 公開/ロールバック | 対象リビジョンを公開。`audit_log` へ記録 | Curator+ |
| GET | /api/revisions | リビジョン履歴 | `slug` でフィルタし、差分表示用に前リビジョンIDも付与 | Editor+ |
| POST | /api/upload | 画像署名URL | Vercel Blob 署名発行。成功後 `Asset` 作成 | Editor+ |
| POST | /api/report | 通報 | ページ/リビジョン/アセットの問題を登録。Rate limit + hCaptcha | Authenticated |
| GET | /api/tags/[tag] | タグ詳細 | タグ説明・関連タグ・ページ一覧 | Public |
| POST | /api/watch | ウォッチ登録 | ページのウォッチ ON/OFF | Authenticated |
| GET | /api/notifications | 通知一覧 | 未読/既読、ウォッチ更新、モデ報告など | Authenticated |

- 全エンドポイントで `zod` による入力検証。
- `middleware.ts` にて `next-safe-middleware`、`csrf`、`rate limit` を適用。
- 監査対象操作（公開/ロールバック/凍結など）は `AuditLog` に記録。

## 4. 認証 / 権限

- NextAuth.js（Discord OAuth）を `/api/auth/[...nextauth]/route.ts` で構成。
- ロールは DB 上の `User.role` で管理。初回ログインで `EDITOR_PENDING` に設定。
- サーバー側では `getCurrentUser()` ヘルパーを `src/server/auth.ts` に用意。
- 権限判定は `assertRole(user, Role.CURATOR)` などのユーティリティで統一。

## 5. フロントエンド構成

### 5.1 テーマ / グローバル設定

- `src/styles/globals.css` に Tailwind ベースのテーマ、金縁（`:before`）、背景グラデーションを定義。
- `tailwind.config.ts` に daisyUI テーマ `vtLight` / `vtDark` を登録。
- `src/lib/theme.ts` でテーマスイッチャー（`prefers-color-scheme` 対応）。
- フォントは Next.js の `next/font/google` で Noto Sans JP / Zen Maru Gothic を読み込み。

### 5.2 ページ別ディレクトリと主要コンポーネント

| Route | ディレクトリ | 主要コンポーネント |
|-------|--------------|--------------------|
| `/` | `src/app/page.tsx` | `<HeroSection>`, `<SearchBar>`, `<RecentUpdatesGrid>`, `<FeaturedPages>`, `<TagCloud>` |
| `/search` | `src/app/search/page.tsx` | `<SearchFilters>`, `<SearchResultList>`, `<InfiniteScrollTrigger>`, `<EmptyState>` |
| `/wiki/[slug]` | `src/app/wiki/[slug]/page.tsx` | `<PageHeader>`, `<InfoBar>`, `<TipTapRenderer>`, `<TableOfContents>`（カテゴリ折りたたみ）, `<InfoboxCard>`, `<RelatedTagCarousel>` |
| `/edit/[slug]` | `src/app/edit/[slug]/page.tsx` | `<EditorShell>`, `<TipTapEditor>`, `<SidebarTabs>`（Infobox / Blocks / Assets / Preview）, `<SaveBar>`, `<DiffSummaryDialog>` |
| `/new` | `src/app/new/page.tsx` | `<PageCreationForm>`, `<TypeSelector>`, `<SlugPreview>` |
| `/history/[slug]` | `src/app/history/[slug]/page.tsx` | `<RevisionList>`, `<RevisionDiffViewer>`, `<RollbackActionBar>` |
| `/tags/[tag]` | `src/app/tags/[tag]/page.tsx` | `<TagOverview>`, `<RelatedTags>`, `<TagPageList>` |
| `/assets` | `src/app/assets/page.tsx` | `<AssetGrid>`, `<AssetFilters>`, `<AssetDetailDrawer>`, `<AssetUsageList>` |
| `/profile` | `src/app/profile/page.tsx` | `<ProfileHeader>`, `<DraftList>`, `<WatchList>`, `<NotificationSettings>` |
| `/admin/moderation` | `src/app/admin/moderation/page.tsx` | `<ReportQueue>`, `<DiffPreviewPanel>`, `<ModerationActions>`, `<RateLimitStats>` |

### 5.3 共通ディレクトリ

- `src/components/ui/`：shadcn/ui ベースのボタン、モーダル、トースト、フォームコンポーネント。
- `src/components/layout/`：`<AppShell>`, `<MainHeader>`, `<SiteFooter>`, `<ThemeSwitcher>`。
- `src/components/wiki/`：Infobox, Timeline, Gallery, YouTubeEmbed, ReferenceList 等 TipTap ノードレンダラー。
- `src/components/editor/`：TipTap 用のメニューバー、BubbleMenu、SlashCommand、ImageUploader。
- `src/components/table-of-contents/`：カテゴリ別折りたたみ TOC コンポーネント。
- `src/features/`：`search/`, `pages/`, `history/`, `moderation/` 等、機能単位の hooks + コンポーネント。

### 5.4 状態管理

- Server Components によるデータフェッチを基本とし、クライアント側は最小限の Zustand ストア（エディタ用 `useEditorStore` など）を使用。
- SWR/React Query を導入せず、`useTransition` と `server actions`（必要箇所）でハンドリング。

## 6. エディタ仕様

- TipTap エディタ拡張：Heading, Bold, Italic, Link, List, Table, Blockquote, Footnote, CodeBlock, HorizontalRule, Image, Gallery, Timeline, Reference。
- 画像アップロードは D&D/ペースト対応、Vercel Blob の一時 URL を取得しプレビュー→保存時に確定。
- オートセーブは `debounce(5000)` で `/api/pages/[id]/revisions` へドラフト投稿。未保存警告を `beforeunload` で表示。
- 権限に応じて即時公開か審査待ちを切り替え。`curator` 以上は「公開」ボタンで `publish` エンドポイント呼び出し。
- 楽観ロック: 最新 `updatedAt` が一致しない場合、競合ダイアログで差分を提示し再取得。
- Heading ノードには `attrs.id`（slugifiedテキスト）、`attrs.level`、`attrs.htmlAttributes['data-category']` を保持。エディタの見出しツールバーにカテゴリセレクト（初期カテゴリ + ユーザー追加分）を設置し、選択内容を `data-category` に即時反映する。翻訳モード（将来の i18n 対応時）はカテゴリは原文のキーを維持し、翻訳対象外として扱う。

### 6.1 見出しカテゴリ解析（サーバーサイド）

1. リビジョン保存時、TipTap JSON (`PageRevision.content`) をサーバーで受領したら、`doc.content` を走査して `type === 'heading'` のノードを抽出。
2. 各見出しで以下の順序でカテゴリを決定する。
   - `node.attrs?.htmlAttributes?.['data-category']` が存在すればそれを採用。
   - 存在しない場合は `node.attrs?.textAlign` 等の他属性は無視し、`node.content` のテキストを連結した上でクライアントと同一のキーワードマッピングテーブルで再判定（整合性チェック）。
   - `node.attrs?.tags`（クライアント側が付与する補助タグ配列）が存在すれば、そこに含まれるカテゴリキーのうち最初のものを採用。
3. 上記のいずれでもカテゴリが確定しない場合は 400 エラーを返し、クライアントに「カテゴリを選択してください」と表示させる（サーバー側でデフォルト付与はしない）。暫定カテゴリとして「その他」を選べるようクライアント UI で案内する。
4. 正常な見出し配列を得たら、`[{ id, text, level, category }]` の形で TOC 生成に利用し、ページ保存時には `PageRevision.content` にも `data-category` を保持する。

## 7. 差分・履歴

- `/history/[slug]` に `RevisionList`（リビジョンID / 編集者 / 要約 / 日時 / ステータス / サイズ）。
- 比較 UI は左右比較とインライン比較の 2 モード。`jsondiffpatch` を TipTap ノード毎に適用し差分をハイライト。
- `curator` 以上は `RollbackActionBar` から対象リビジョンを指定して `/api/pages/[id]/publish` を呼び出す。

## 8. 通知 / モデレーション

- `/api/report` で登録された通報は `ReportQueue` にリスト表示。添付差分プレビューから `Resolve` / `Rollback` / `FreezeUser`（将来）を選択。
- Rate limit 状況、IP/UA メトリクスはログベースで `RateLimitStats` に表示。
- ウォッチ通知はページ更新時に Webhook/メール（後日）を送信。通知一覧は `/profile` で管理。

## 9. アクセシビリティ / i18n

- `aria-label` とフォーカスリングを全操作要素に適用。
- 主要テキストはコントラスト比 4.5:1 を満たすよう Tailwind カスタムカラーを調整。
- `prefers-reduced-motion` を検知し、モーション系コンポーネントではアニメーションを抑制。
- 国際化は `src/i18n/{ja,en}.ts` に辞書を配置し、`<LocaleProvider>` で配布。

## 10. パフォーマンス / デプロイ

- ホーム・検索は ISR + キャッシュタグ（再検証は編集・公開時に失効）。
- `next/image` + Vercel Optimizer で画像最適化。ギャラリーは遅延読み込み。
- 重要操作に Skeleton / Loading UI を提供し、`Suspense` で段階表示。
- Vercel へデプロイ。環境変数例: `DATABASE_URL`, `NEXTAUTH_SECRET`, `DISCORD_CLIENT_ID/SECRET`, `BLOB_READ_WRITE_TOKEN`, `SENTRY_DSN`。

## 11. テスト戦略

- 単体テスト: Jest + React Testing Library（主要 UI、ロール制御）。
- API テスト: Vitest もしくは Jest で Route Handler をモックし zod スキーマ検証。
- E2E テスト: Playwright（閲覧/検索/編集/公開/通報フロー）。
- アクセシビリティ: `@axe-core/playwright` で主要ページを検査。

## 12. 実装ロードマップ

1. 認証・DB 基盤構築（NextAuth, Prisma）
2. ページ閲覧 & 検索 MVP（TipTap Renderer, TOC, タグ）
3. エディタ & リビジョンフロー実装（下書き、承認、公開）
4. テーマ適用・キャラクターページ特別レイアウト・Hero セクション演出
5. 差分ビュー / 履歴 UI 強化 + モデレーション操作
6. アセット管理・通報キュー・通知設定
7. Meilisearch 導入、日本語検索強化、将来の協調編集（Yjs/Liveblocks 検討）

