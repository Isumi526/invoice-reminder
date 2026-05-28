# 03. アーキテクチャ

## 全体図

```
                    ┌──────────────────────┐
                    │  ユーザー(ブラウザ)│
                    └──────────┬───────────┘
                               │ HTTPS
                               ▼
                    ┌──────────────────────┐
                    │  Vercel(Next.js 15) │
                    │  - App Router         │
                    │  - Server Actions     │
                    │  - Edge Middleware    │
                    └──┬───────┬───────┬───┘
                       │       │       │
              ┌────────┘       │       └──────────┐
              │                │                  │
              ▼                ▼                  ▼
      ┌──────────────┐ ┌──────────────┐  ┌──────────────┐
      │  Supabase    │ │  SendGrid    │  │   Stripe     │
      │  - Postgres  │ │  - Send API  │  │   - Billing  │
      │  - Auth      │ │  - Templates │  │   - Webhook  │
      │  - Storage   │ │  - Webhook   │  └──────────────┘
      │  - Edge Func │ └──────┬───────┘
      └──────┬───────┘        │
             │                │ (bounce/open/click events)
             │                ▼
             │        ┌──────────────────┐
             └───────▶│  Vercel API      │
                      │  /api/webhooks   │
                      └──────────────────┘

         ┌────────────────────────────────┐
         │  Vercel Cron(毎日 09:00 JST) │
         │  → /api/cron/daily-reminder    │
         │  → 期限超過の検出と送信予約      │
         └────────────────────────────────┘
```

## レイヤ責務

### Client Layer(`app/` `components/`)

- React Server Components(RSC)を基本とし、必要な箇所のみ Client Component
- 状態管理は Server Actions + `useFormState` / `useOptimistic` を中心に
- グローバル状態が必要な場合のみ Zustand を最小限導入
- フォームは `react-hook-form` + `zod` の組み合わせ

### Server Action Layer(`server/`)

- ビジネスロジックの中心、ドメイン単位でファイル分割
- 各 action は以下の構造で記述:

```typescript
'use server';

import { z } from 'zod';
import { requireAuth } from '@/lib/auth';
import { Result, ok, err } from '@/lib/result';

const schema = z.object({
  // 入力スキーマ
});

export async function actionName(
  input: z.infer<typeof schema>
): Promise<Result<ReturnType, AppError>> {
  const auth = await requireAuth();
  if (!auth.ok) return auth;

  const parsed = schema.safeParse(input);
  if (!parsed.success) return err({ kind: 'validation', issues: parsed.error });

  // ビジネスロジック
  // DB アクセスは lib/supabase 経由

  return ok(result);
}
```

### Data Layer(`lib/supabase/`)

- 3 種類のクライアントを使い分け:
  - `createClient()` - ブラウザ用(anon key)
  - `createServerClient()` - Server Components / Actions 用(anon key + cookie)
  - `createAdminClient()` - Edge Functions / API Routes の特権操作用(service role key)
- `service_role` キーは絶対にブラウザに渡さない
- クエリは Supabase JS SDK を使い、SQL は migration ファイルでのみ書く

### Integration Layer(`lib/sendgrid/` `lib/stripe/`)

- 外部サービス連携の境界として独立させる
- 各サービスのレスポンスは独自型に変換してから呼び出し元に返す
- リトライ・タイムアウト・サーキットブレーカーは将来的に検討(Phase 1 では最低限)

## 主要フロー

### A. 請求書登録 → リマインドスケジュール作成

```
[ユーザー]
  ↓ フォーム送信
[Server Action: createInvoice]
  ↓ Zod 検証
  ↓ Supabase: insert invoices
  ↓ scheduleReminders(invoice) を呼ぶ
[ヘルパー関数: scheduleReminders]
  ↓ 期限から 5 つの reminder レコードを scheduled で作成
  ↓ Supabase: bulk insert reminders
  ↓
[ユーザーに「登録完了」を返す]
```

### B. 毎日 09:00 のリマインド送信(Vercel Cron)

```
[Vercel Cron]
  ↓ GET /api/cron/daily-reminder
  ↓ X-Cron-Secret ヘッダで認証
[Cron Handler]
  ↓ Supabase: select reminders where scheduled_at <= now() and status = 'scheduled'
  ↓ 各 reminder について:
    ↓ invoices と organizations を JOIN して文面を組み立て
    ↓ SendGrid: send mail
    ↓ Supabase: update reminders set status='sent', sent_at=now(), sendgrid_message_id=...
  ↓ 完了レスポンス
```

### C. SendGrid Event Webhook(バウンス・開封・クリック)

```
[SendGrid]
  ↓ POST /api/webhooks/sendgrid
  ↓ 署名検証(公開鍵で ed25519)
[Webhook Handler]
  ↓ イベント配列をパース
  ↓ 各イベントについて:
    ↓ sendgrid_message_id で reminder を特定
    ↓ events JSON に追記
    ↓ status を必要に応じて更新(bounced/opened/clicked)
  ↓ 200 を返す(SendGrid のリトライ防止)
```

### D. 入金確認(手動)→ リマインド停止

```
[ユーザー]
  ↓ 「入金確認済み」ボタンクリック
[Server Action: markAsPaid]
  ↓ 確認ダイアログで二段階確認
  ↓ Supabase トランザクション:
    ↓ invoices: status='paid', paid_at=now()
    ↓ reminders: status='cancelled' where invoice_id=? and status='scheduled'
  ↓ audit_logs に記録
  ↓
[ユーザーに完了通知]
```

### E. Stripe Webhook(サブスク状態同期)

```
[Stripe]
  ↓ POST /api/webhooks/stripe
  ↓ 署名検証
[Webhook Handler]
  ↓ event.type に応じて分岐:
    - customer.subscription.created → organizations.plan 更新
    - customer.subscription.updated → 同上
    - customer.subscription.deleted → plan='lite' or 'trial' に降格
    - invoice.payment_failed → 督促状態に
  ↓ Supabase 更新
```

## エラーハンドリング戦略

### Result 型

例外を投げず、Result 型で返す:

```typescript
type Result<T, E> = { ok: true; data: T } | { ok: false; error: E };

type AppError =
  | { kind: 'auth'; message: string }
  | { kind: 'validation'; issues: z.ZodError }
  | { kind: 'not_found'; resource: string }
  | { kind: 'forbidden'; reason: string }
  | { kind: 'external'; service: string; cause: unknown }
  | { kind: 'unknown'; cause: unknown };
```

### エラー UI

- Server Action がエラーを返したら `useFormState` で受け取り、フォーム上に表示
- `error.tsx` は最終的なフォールバック、Sentry に通知
- `not-found.tsx` で 404 を統一

## ログ・監視

- **アプリログ**: Vercel Logs(短期保管)
- **エラー追跡**: Sentry(本番のみ、PII マスキング設定)
- **メトリクス**: Vercel Analytics(コア Web バイタル)
- **DB クエリ**: Supabase Dashboard でスロークエリ監視
- **アラート**: Sentry → Slack(全力AIラボのワークスペース)

## パフォーマンス目標

- TTFB < 500ms(p95)
- LCP < 2.5s
- ダッシュボード初回読込 < 3s(50 件以下の請求書一覧時)
- リマインドメール送信スループット: 1000 件/分(SendGrid の制限内)

## セキュリティ境界

- ブラウザに渡してよいもの: `NEXT_PUBLIC_*` のみ
- Server Components / Actions のみ: `SUPABASE_SERVICE_ROLE_KEY` `SENDGRID_API_KEY` `STRIPE_SECRET_KEY`
- Webhook エンドポイントは必ず署名検証
- Cron エンドポイントは `CRON_SECRET` ヘッダで認証
- RLS で組織間データ漏洩を多層防御

## 環境差分

| 環境 | DB | メール送信 | Stripe |
|---|---|---|---|
| local | Supabase ローカル(Docker) | Mailpit(ローカル SMTP) | テストモード |
| staging | Supabase 本番(別プロジェクト) | SendGrid Sandbox | テストモード |
| production | Supabase 本番 | SendGrid 本番 IP | 本番モード |

## 将来の拡張ポイント

- **キュー導入**: 送信量が増えたら Trigger.dev / Inngest 導入を検討
- **マルチリージョン**: 当面は東京リージョン固定
- **モバイルアプリ**: Phase 3 以降、必要なら Expo で別途構築
- **API 公開**: 他社 SaaS との連携用、認証は Supabase JWT を流用
