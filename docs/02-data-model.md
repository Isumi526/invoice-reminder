# 02. データモデル

## ERD(Phase 1)

```
┌─────────────────┐
│ organizations   │  企業(マルチテナント単位)
├─────────────────┤
│ id (uuid, PK)   │
│ name            │
│ representative  │
│ address         │
│ phone           │
│ stripe_customer │
│ plan            │
│ created_at      │
└────────┬────────┘
         │
         │ 1:N
         ▼
┌─────────────────┐         ┌─────────────────┐
│ memberships     │   N:1   │ profiles        │
├─────────────────┤────────▶├─────────────────┤
│ id              │         │ id (=auth.uid)  │
│ org_id (FK)     │         │ display_name    │
│ user_id (FK)    │         │ email           │
│ role            │         │ phone           │
│ created_at      │         │ created_at      │
└─────────────────┘         └─────────────────┘
         │
         │ (org に紐づく)
         ▼
┌─────────────────┐         ┌─────────────────┐
│ debtors         │ 1:N     │ invoices        │
├─────────────────┤◀────────┤─────────────────│
│ id              │         │ id              │
│ org_id (FK)     │         │ org_id (FK)     │
│ company_name    │         │ debtor_id (FK)  │
│ contact_name    │         │ invoice_number  │
│ email           │         │ subject         │
│ phone           │         │ amount          │
│ address         │         │ issued_at       │
│ notes           │         │ due_at          │
└─────────────────┘         │ paid_at         │
                            │ status          │
                            │ late_fee_rate   │
                            │ pdf_path        │
                            └────────┬────────┘
                                     │
                                     │ 1:N
                                     ▼
                            ┌─────────────────┐
                            │ reminders       │  送信履歴
                            ├─────────────────┤
                            │ id              │
                            │ invoice_id (FK) │
                            │ stage           │  1〜5
                            │ scheduled_at    │
                            │ sent_at         │
                            │ subject         │
                            │ body            │
                            │ from_email      │
                            │ to_email        │
                            │ sendgrid_msg_id │
                            │ status          │  pending|sent|bounced|opened|clicked
                            │ events (jsonb)  │
                            └─────────────────┘
```

## テーブル定義

### `organizations`

マルチテナントの最上位単位。1 企業 = 1 organization。

```sql
create table organizations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  representative_name text,
  postal_code text,
  address text,
  phone text,
  email text,
  default_from_email text,
  default_late_fee_rate numeric(5,3) default 0.030,  -- 年率 3%
  stripe_customer_id text unique,
  plan text not null default 'lite' check (plan in ('lite','standard','pro','trial')),
  trial_ends_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

### `profiles`

Supabase Auth ユーザーの拡張情報。`auth.users` と 1:1。

```sql
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  display_name text,
  email text not null,
  phone text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

### `memberships`

ユーザーと組織の N:M 中間テーブル。

```sql
create table memberships (
  id uuid primary key default gen_random_uuid(),
  organization_id uuid not null references organizations(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  role text not null default 'member' check (role in ('owner','admin','member')),
  created_at timestamptz not null default now(),
  unique(organization_id, user_id)
);
```

### `debtors`

請求先(取引先)マスタ。

```sql
create table debtors (
  id uuid primary key default gen_random_uuid(),
  organization_id uuid not null references organizations(id) on delete cascade,
  company_name text not null,
  contact_name text,
  email text not null,
  phone text,
  postal_code text,
  address text,
  notes text,
  is_blocked boolean not null default false,  -- 取引停止フラグ
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index idx_debtors_org on debtors(organization_id);
create index idx_debtors_email on debtors(email);
```

### `invoices`

請求書本体。

```sql
create type invoice_status as enum (
  'draft',         -- 下書き
  'active',        -- リマインド対象
  'paid',          -- 入金確認済み
  'cancelled',     -- 取消
  'escalated'      -- ステージ 5 到達(専門家送客)
);

create table invoices (
  id uuid primary key default gen_random_uuid(),
  organization_id uuid not null references organizations(id) on delete cascade,
  debtor_id uuid not null references debtors(id) on delete restrict,
  invoice_number text not null,
  subject text not null,                       -- 件名(工事名等)
  amount numeric(12,0) not null check (amount > 0),
  tax_amount numeric(12,0) not null default 0,
  total_amount numeric(12,0) generated always as (amount + tax_amount) stored,
  issued_at date not null,
  due_at date not null,
  paid_at timestamptz,
  partial_paid_amount numeric(12,0) not null default 0,
  status invoice_status not null default 'active',
  late_fee_rate numeric(5,3) not null default 0.030,
  pdf_storage_path text,
  memo text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique(organization_id, invoice_number)
);

create index idx_invoices_org_status on invoices(organization_id, status);
create index idx_invoices_due on invoices(due_at) where status = 'active';
```

### `reminders`

リマインドメールの送信履歴・予定。

```sql
create type reminder_stage as enum ('s1','s2','s3','s4','s5');
create type reminder_status as enum (
  'scheduled',     -- 送信予定
  'queued',        -- 送信キューに投入
  'sent',          -- 送信完了
  'failed',        -- 送信失敗
  'cancelled',     -- 入金等で取消
  'bounced',       -- バウンス
  'opened',        -- 開封(精度限界あり)
  'clicked'        -- リンククリック
);

create table reminders (
  id uuid primary key default gen_random_uuid(),
  invoice_id uuid not null references invoices(id) on delete cascade,
  organization_id uuid not null references organizations(id) on delete cascade,
  stage reminder_stage not null,
  scheduled_at timestamptz not null,
  sent_at timestamptz,
  subject text not null,
  body_text text not null,
  body_html text,
  from_email text not null,
  from_name text not null,
  to_email text not null,
  cc text[],
  sendgrid_message_id text,
  status reminder_status not null default 'scheduled',
  events jsonb not null default '[]'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index idx_reminders_scheduled on reminders(scheduled_at) where status = 'scheduled';
create index idx_reminders_invoice on reminders(invoice_id);
create index idx_reminders_sendgrid on reminders(sendgrid_message_id);
```

### `audit_logs`

監査ログ。誰がいつ何をしたかを記録。

```sql
create table audit_logs (
  id uuid primary key default gen_random_uuid(),
  organization_id uuid not null references organizations(id) on delete cascade,
  actor_id uuid references auth.users(id) on delete set null,
  action text not null,                        -- 'invoice.create' 'reminder.send' 等
  target_type text,
  target_id uuid,
  metadata jsonb not null default '{}'::jsonb,
  ip_address inet,
  user_agent text,
  created_at timestamptz not null default now()
);

create index idx_audit_org_created on audit_logs(organization_id, created_at desc);
```

## RLS ポリシー

**全テーブルで RLS を有効化する。ポリシー未設定の table は本番に出さない。**

### 基本パターン

```sql
-- 自分が所属する organization のデータのみ閲覧可
create policy "members can read own org data"
on debtors for select
using (
  organization_id in (
    select organization_id from memberships where user_id = auth.uid()
  )
);

-- 所属組織への書き込みは owner / admin のみ
create policy "admins can write"
on debtors for insert
with check (
  organization_id in (
    select organization_id from memberships
    where user_id = auth.uid() and role in ('owner','admin')
  )
);
```

### `reminders` の特別ルール

- `sent_at` がセットされた reminder は **update / delete 禁止**(監査証跡として保護)
- `cancelled` への状態変更のみ許可

```sql
create policy "no edit on sent reminders"
on reminders for update
using (status not in ('sent','bounced','opened','clicked'));
```

## マイグレーション運用

- ファイル名: `supabase/migrations/YYYYMMDDHHMMSS_description.sql`
- 1 マイグレーション = 1 トランザクション
- ロールバック SQL は同名 `.down.sql` に保存(任意)
- 本番適用前に必ず staging で動作確認
- 大規模なデータ移行は migration とは別の手動スクリプトで実施

## 型生成

```bash
pnpm supabase gen types typescript --linked > types/database.ts
```

CI で自動生成 → コミット差分があれば PR をブロックする運用にする。

## Phase 2 で追加予定のテーブル

- `payments` - 入金記録(マネフォ連携で自動取り込み)
- `bank_accounts` - 振込先口座マスタ
- `integrations` - 外部サービス連携設定(マネフォ・freee 等)
- `templates` - ユーザー定義の文面テンプレート
- `legal_partners` - 弁護士・司法書士の提携先マスタ
