# 07. 開発ワークフロー

## ブランチ戦略

シンプルな GitHub Flow ベース。

```
main         ← 本番(Vercel Production にデプロイ)
  ↑
develop      ← 統合用(Vercel Preview に常時デプロイ)
  ↑
feature/*    ← 機能開発
fix/*        ← バグ修正
chore/*      ← 設定変更・依存更新
```

### ルール

- `main` への直接 push 禁止、必ず PR 経由
- PR には `develop` ベースで作成、CI パス + Claude Code のレビュー or 亥角さん自身の確認後にマージ
- `develop` → `main` は計画的にマージ(週 1〜2 回程度を目安)
- ホットフィックスは `main` から切って `main` と `develop` 両方にマージ

## コミットメッセージ

Conventional Commits を採用。

```
<type>(<scope>): <subject>

<body>

<footer>
```

| type | 用途 |
|---|---|
| `feat` | 新機能 |
| `fix` | バグ修正 |
| `docs` | ドキュメント |
| `style` | フォーマット |
| `refactor` | 動作変わらないリファクタ |
| `perf` | パフォーマンス改善 |
| `test` | テスト追加・修正 |
| `chore` | 雑務(依存更新等) |
| `revert` | リバート |

例:

```
feat(invoices): add bulk import via CSV

CSV ファイルからの請求書一括登録機能を追加。
列マッピングは初回 UI で設定、2 回目以降は記憶する。

Closes #42
```

## PR テンプレート

```markdown
## 概要
<!-- 何を、なぜ変更したか -->

## 変更内容
- [ ] 変更点 1
- [ ] 変更点 2

## 影響範囲
<!-- 既存機能への影響、DB マイグレーション有無 -->

## テスト
- [ ] ユニットテスト追加
- [ ] 手動動作確認(staging)
- [ ] RLS ポリシー検証(該当する場合)

## スクリーンショット
<!-- UI 変更がある場合 -->

## レビュー観点
<!-- 特に見てほしいポイント -->
```

## テスト戦略

### Phase 1 では「重要箇所のみ」徹底

| レイヤ | テスト種別 | 必須度 |
|---|---|---|
| Server Actions | ユニットテスト | 必須 |
| ビジネスロジック(遅延損害金計算等) | ユニットテスト | 必須 |
| メール送信フロー | 統合テスト(SendGrid モック) | 必須 |
| Webhook ハンドラ | 統合テスト | 必須 |
| UI コンポーネント | スナップショット | 任意 |
| E2E | Playwright(認証〜請求書登録〜送信完了の 1 シナリオ) | Phase 1 末で 1 本 |

### テストツール

- **ユニット・統合**: Vitest
- **E2E**: Playwright
- **モック**: MSW(SendGrid / Stripe / マネフォの API モック)

### カバレッジ目標

- ビジネスロジック: 80% 以上
- 全体: 計測のみ、強制はしない(過剰テストでスピードを落とさない)

## CI / CD

### GitHub Actions

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main, develop]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
```

### マイグレーション CI

`.github/workflows/migrate.yml`:

```yaml
name: Apply Migrations

on:
  push:
    branches: [main]
    paths: ['supabase/migrations/**']

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: supabase/setup-cli@v1
      - run: supabase db push --linked
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
          SUPABASE_DB_PASSWORD: ${{ secrets.SUPABASE_DB_PASSWORD }}
```

### Vercel デプロイ

- `main` → Production 自動
- `develop` → Preview(staging URL)
- PR → 個別 Preview URL

## ローカル開発環境

### 初回セットアップ

```bash
# Node.js / pnpm 準備
nvm install 22
npm install -g pnpm@9

# リポジトリ
git clone <repo>
cd <repo>
pnpm install

# Supabase ローカル(Docker 必須)
pnpm supabase start
pnpm supabase db reset  # マイグレーション + seed 適用

# 環境変数
cp .env.example .env.local
# Supabase ローカルの URL/anon key を貼る(supabase status の出力)

# Mailpit(ローカルメール受信)
docker run -d --name mailpit -p 1025:1025 -p 8025:8025 axllent/mailpit

# 開発サーバー
pnpm dev
```

### 日次起動

```bash
pnpm supabase start
pnpm dev
```

## コードレビュー観点

セルフレビュー or Claude Code への依頼時のチェック項目:

### 機能・設計

- [ ] CLAUDE.md と docs/ の方針に沿っているか
- [ ] スコープ外の変更が紛れていないか
- [ ] 既存の設計パターンと整合しているか

### 型安全性

- [ ] `any` を使っていないか
- [ ] Zod スキーマで境界バリデーションしているか
- [ ] DB 型は最新の自動生成と一致しているか

### セキュリティ

- [ ] 新規テーブルに RLS が設定されているか
- [ ] service_role キーがクライアントに漏れていないか
- [ ] Webhook の署名検証が入っているか
- [ ] XSS / SQL インジェクションの余地はないか

### パフォーマンス

- [ ] N+1 クエリになっていないか
- [ ] 不要な Client Component 化がないか
- [ ] 大量データ表示時のページネーション

### UX

- [ ] エラー時の UI が用意されているか
- [ ] 読込中の表示があるか
- [ ] 重要操作に確認ダイアログがあるか

## Claude Code との協働

### 指示の出し方

- **小さく切る**: 「請求書登録フォームを作って」より「請求書登録フォームの取引先選択ステップを作って」
- **コンテキストを与える**: 「`docs/04-reminder-flow.md` を読んで」と明示する
- **完了条件を明示**: 「ユニットテストも追加、`pnpm test` がパスすること」
- **設計判断が必要な場面では先に確認**: 「実装方針として A 案 / B 案があるが、どちらにする?」と聞かせる

### Claude Code 側で守ること

- 必ず CLAUDE.md と関連 docs を読んでから着手
- 不明点は推測せず質問する
- 大きな構造変更前に設計案を提示
- DB スキーマ変更時は migration と docs/02 を同時更新
- パッケージ追加時は理由を説明
- テストを書く

### 禁止事項

- `.env.local` に値を勝手に書き込まない(変数名のみ `.env.example` に追加)
- 本番 DB への直接操作
- マイグレーションをスキップした schema 変更
- ライセンス不明・GPL のライブラリ導入
- 既存テストを「とりあえず通すため」に書き換える

## リリースフロー

### Phase 1 中の運用

1. feature ブランチで開発 → PR
2. CI パス確認 → develop へマージ
3. Preview URL で β顧客にも確認してもらう
4. 週次で develop → main へマージ → 本番デプロイ
5. リリースノートを Notion / GitHub Release に記載

### バージョニング

- セマンティックバージョニング: `MAJOR.MINOR.PATCH`
- Phase 1 リリース時に `v0.1.0`、有料リリース時に `v1.0.0`
- Git タグで管理

## 障害対応

- 異常を検知したら、まず `incidents/YYYY-MM-DD.md` に時系列で記録開始
- ロールバックは Vercel の「Promote previous deployment」で実施
- DB ロールバック: マイグレーション逆順の down SQL を手動実行(自動化は Phase 2 以降)
- β顧客には Slack / メールで即連絡
