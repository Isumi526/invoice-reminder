# CLAUDE.md

このファイルは Claude Code が本プロジェクトで作業する際の最上位指示書です。
作業開始時は必ず本ファイルと `docs/` 配下を読み、コンテキストを把握してください。

## プロジェクト概要

**プロダクト名(仮)**: **Tonton(トントン)**
**GitHub リポジトリ名**: `invoice-reminder`(技術名、プロダクト名と分離)
**リポジトリ公開設定**: **Public**(機密情報の扱いに最大限の注意が必要)
**プロダクト種別**: BtoB SaaS / 未払い請求書の段階的リマインドシステム
**ターゲット**: 建設業の中小事業者(初期)、将来的に業種横断
**運営**: 合同会社スティズム / 全力AIラボ
**β顧客**: 株式会社シード(レベニューシェアモデル、別プロダクト sido の β顧客でもある)

> **⚠️ Public リポジトリです**
> 本リポジトリは GitHub Public で公開されています。
> 機密情報・顧客情報・営業秘密の取り扱いは `docs/10-secrets-and-public-repo.md` を
> **必ず最初に読み、すべての作業で遵守すること。**
>
> - API キー・パスワード・トークンを **絶対にコミットしない**
> - 顧客名・取引先名・実在の金額を Issue / PR / コミットに書かない
> - テストデータは架空企業のみ(`株式会社サンプル` `test@example.com` 等)
> - シークレットは `.env.local`(Git 管理外)または Vercel / GitHub Secrets で管理

> **命名規約について**
> - **Tonton**: 顧客向け呼称(LP・メール文面・契約書・UI 上のロゴ表記)
> - **invoice-reminder**: 技術的識別子(GitHub リポジトリ・Vercel プロジェクト・Supabase プロジェクト・パッケージ名)
> - コード内のクラス・関数名は機能ベース(`Invoice` `Reminder` 等)で、ブランド名は埋め込まない
> - プロダクト名が正式決定したら一括 rename 可能な設計を維持する

## ビジネスコンテキスト

### 解決する課題

建設業を中心とした中小事業者の以下の悩みを解決する:

- 取引先からの請求書の未払いが頻発する
- 債権回収会社に依頼しても回収費用でほぼ消える
- 自社での催促業務の工数が重い
- 踏み倒されると精神的に消耗する

### 提供価値

- 期限超過から自動でリマインドメールを段階的に送信
- 送信元はユーザー名義(自社情報を含む)で、ボット感を排除
- 超過日数に応じた遅延損害金計算・文面の自動エスカレーション
- 入金確認後の即時停止、誤送信防止のセミオート設計

### やらないこと(プロダクトの境界)

以下は明確にスコープ外とする:

- 債権回収の代行(弁護士法72条 非弁行為に抵触する可能性)
- 訴状作成支援(法律事務に該当)
- 完全自動催促(人間の承認なしの送信)
- 電子内容証明の代行送信(日本郵便の規約上難しい)

## 関連プロダクト(sido)との関係

**sido** は β顧客(株式会社シード)向けに別途開発している
**施工台帳管理 SaaS** です。Tonton とは以下の関係:

- **プロダクトとして完全独立**: Tonton は単独で価値が成立し、sido がなくても運用可能
- **コードベース**: 完全分離(別リポジトリ・別 Vercel プロジェクト・別 Supabase プロジェクト)
- **ブランド・販売**: Tonton は単独プロダクトとしてマーケティング、sido とのバンドル販売は将来のオプション
- **顧客接点**: シード社は両プロダクトを契約しているが、他顧客は単体契約が前提

### 将来的な技術連携(オプション)

両プロダクトを契約する顧客向けに、以下の連携を将来検討する余地を残す:

- **共通 organization_id**: 同一企業の場合、両側で同じ UUID を使う設計を許容
- **Webhook 連携**: sido の「工事完了」イベント → Tonton の請求書発行待ちキューへの投入
- **SSO**: 両方のアカウント間でのトークン共有

ただし、これらは **Phase 2 以降の連携機能** であり、Phase 1 では sido を意識した
設計は一切行わない。Tonton 単独で完結する設計を最優先する。

### バックエンドを別 Supabase プロジェクトにする理由

スタックは Supabase で揃えるが、プロジェクトは分離する:

- 認証ユーザー(`auth.users`)が混ざらない
- RLS ポリシーが複雑化しない
- 障害・マイグレーションの影響範囲が独立
- DB プランの容量・料金が独立

詳細は `docs/01-product-strategy.md` 参照。

## 技術スタック

| レイヤ | 採用技術 | 理由 |
|---|---|---|
| フロントエンド | Next.js 15(App Router) | Vercel との親和性、Server Actions、Streaming |
| 言語 | TypeScript(strict) | 型安全、AI 補完の精度確保 |
| UI | Tailwind CSS v4 + shadcn/ui | カスタマイズ性、Radix ベースのアクセシビリティ |
| バックエンド | **Supabase**(Postgres + RLS + Edge Functions + Storage) | 後述の選定理由参照 |
| 認証 | Supabase Auth(Email + Magic Link) | パスワードレス前提、零細事業者向けの導入容易性 |
| メール送信 | SendGrid(Dynamic Templates) | ドメイン認証、Event Webhook によるバウンス処理 |
| 決済 | Stripe(Subscription) | 月額課金、日本円対応 |
| ホスティング | Vercel(Pro プラン想定) | Next.js 最適化、プレビュー環境 |
| 監視 | Sentry + Vercel Analytics | エラー追跡、コア Web バイタル |
| OCR(Phase 2) | Google Cloud Vision API | 請求書 PDF からの自動抽出 |
| 会計連携(Phase 2) | マネーフォワード API / freee API | 入金消込の自動化 |

### バックエンド選定理由(Firebase ではなく Supabase)

検討した上で **Supabase を採用**。理由:

1. **要件との相性**: 集計クエリ・トランザクション・JOIN が多いシステム特性は
   Postgres + SQL が素直。Firestore はドキュメント指向で設計が歪む
2. **法務・監査要件**: 監査ログの不変性・送信履歴の編集禁止・物理削除といった
   要件は RLS + Postgres 制約で素直に実装できる
3. **sido との将来連携**: sido も Supabase で構築されているため、
   `organization_id` 体系の統一・型生成・Edge Functions の知見共有が容易
4. **スタック統一**: 全力AIラボ全体で Nuxt/Next + Supabase に揃えることで、
   AI 駆動開発の文脈再利用と知見の積み上げを優先

Firebase は本プロダクトの要件(集計重視・監査必須・複雑フィルタ)に対しては
オーバーヘッドが大きく、リアルタイム同期やオフライン対応の優位性も
本プロダクトでは活かせないため見送り。

### バージョン固定方針

- Node.js: 22.x LTS
- pnpm: 9.x
- 主要パッケージはバージョンレンジではなく固定バージョン(`"^"` `"~"` 禁止)

## ディレクトリ構成

```
.
├── CLAUDE.md                    # 本ファイル
├── docs/                        # 設計ドキュメント(下記参照)
├── app/                         # Next.js App Router
│   ├── (auth)/                  # 認証関連ルート
│   ├── (dashboard)/             # ログイン後のメインアプリ
│   ├── api/                     # API Routes(Webhook 受信用のみ)
│   └── layout.tsx
├── components/                  # 共通 UI コンポーネント
│   ├── ui/                      # shadcn/ui の生成物
│   └── features/                # ドメイン固有のコンポーネント
├── lib/
│   ├── supabase/                # Supabase クライアント(client/server/admin)
│   ├── sendgrid/                # メール送信ロジック
│   ├── stripe/                  # 決済処理
│   └── utils/                   # 汎用ユーティリティ
├── server/                      # Server Actions
│   ├── invoices/
│   ├── reminders/
│   └── billing/
├── supabase/
│   ├── migrations/              # SQL マイグレーション
│   ├── functions/               # Edge Functions(cron / webhook)
│   └── seed.sql                 # 開発環境用初期データ
├── types/
│   ├── database.ts              # Supabase の型自動生成
│   └── domain.ts                # ドメイン型定義
├── tests/
│   ├── unit/
│   └── e2e/
└── package.json
```

## 開発ルール

### 必読の設計ドキュメント

新しい機能に着手する前に、関連する `docs/` を必ず読む:

1. `docs/00-overview.md` - 全体像
2. `docs/01-product-strategy.md` - プロダクト戦略・スコープ判断基準
3. `docs/02-data-model.md` - DB スキーマとリレーション
4. `docs/03-architecture.md` - システムアーキテクチャ
5. `docs/04-reminder-flow.md` - リマインドステージ・文面ロジック
6. `docs/05-security-compliance.md` - 法務・セキュリティ要件
7. `docs/06-ui-design.md` - UI/UX ガイドライン
8. `docs/07-development-workflow.md` - Git フロー・テスト・デプロイ
9. `docs/08-phase-roadmap.md` - Phase 1/2/3 のスコープ
10. `docs/09-glossary.md` - ドメイン用語集
11. **`docs/10-secrets-and-public-repo.md` - シークレット管理・Public リポジトリ運用(全作業で遵守)**

### コーディング規約

- **TypeScript strict mode**: `any` 禁止、`unknown` で受けて型ガード
- **Server Actions 優先**: API Routes は Webhook 受信のみ、それ以外は Server Actions
- **RLS 必須**: 全テーブルで Row Level Security を有効化、ポリシー未設定の table 作成禁止
- **Zod でバリデーション**: Server Actions の入力は全て Zod スキーマで検証
- **エラーハンドリング**: try-catch ではなく Result 型(`{ ok: true, data } | { ok: false, error }`)
- **ファイル長**: 1 ファイル 300 行以内、超える場合は分割を検討
- **コメント**: 「なぜ」を書く、「何を」はコードで表現

### Claude Code への指示

- **読む順番**: CLAUDE.md → `docs/10-secrets-and-public-repo.md` → 該当する docs/ → 既存コード → 実装
- **大きな変更前**: 必ずユーザーに設計案を提示して承認を得る
- **DB スキーマ変更**: マイグレーションファイル作成 + `docs/02-data-model.md` の更新を同時に
- **新規パッケージ追加**: 必ず理由を説明、軽量な代替案がないか検討
- **テスト**: 新規 Server Action には最低 1 つのユニットテスト
- **シークレット**: `.env.local` 直接編集禁止、`.env.example` には **変数名のみ** 追加(値は書かない)
- **Public リポジトリ前提**: 顧客名・取引先名・実在の金額・実在のメアドを **コード・ドキュメント・コミットメッセージのいずれにも書かない**
- **テストデータ**: 必ず架空企業(`株式会社サンプル建設`、`test@example.com`、`03-0000-0000` 等)を使用
- **シークレット混入の検知**: ユーザーが誤って API キー等を貼り付けた場合、コードに反映する前に警告し、環境変数化を提案する

## 環境変数

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=          # サーバー側のみ、絶対に公開しない

# SendGrid
SENDGRID_API_KEY=
SENDGRID_FROM_EMAIL=                # ユーザー設定で上書き
SENDGRID_WEBHOOK_PUBLIC_KEY=        # Event Webhook 署名検証

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# App
NEXT_PUBLIC_APP_URL=
CRON_SECRET=                         # Vercel Cron の認証用
```

## デプロイ

- **本番**: Vercel Production(main ブランチ自動デプロイ)
- **ステージング**: Vercel Preview(develop ブランチ + PR 単位)
- **データベースマイグレーション**: GitHub Actions で staging → production の順に適用
- **Vercel Cron**: 毎日 9:00 JST に `/api/cron/daily-reminder` を叩く

## 連絡先

- 開発主: 亥角 貴治(全力AIラボ)
- β顧客: 株式会社シード CEO
- 緊急時: ismsn526@gmail.com

---

**重要**: このファイルはプロジェクトの全体地図です。
個別の詳細は必ず `docs/` 配下を参照してください。
迷ったら `docs/00-overview.md` から読み始めてください。

**Public リポジトリ警告**: 本リポジトリは GitHub Public で公開されています。
コミット・Issue・PR・ドキュメント作成時は、`docs/10-secrets-and-public-repo.md` の
ルールを **必ず守ってください**。シークレット流出は致命的な影響を及ぼします。
