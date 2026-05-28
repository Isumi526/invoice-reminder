# 10. シークレット管理と Public リポジトリ運用

本リポジトリは **GitHub Public** で公開する前提で運用します。
このドキュメントは、機密情報・営業秘密・顧客情報を絶対に流出させないための運用ルールです。

**この文書のルールは、CLAUDE.md と並んで最優先で守ること。**
迷ったら「コミットしない」「Issue に書かない」を選ぶ。

---

## 1. 絶対にコミットしてはいけないもの

### 認証情報・API キー

以下は **1 文字でもコミットしたら即座にローテーション** が必要:

- Supabase の `service_role` キー、anon キー(URL は公開可)
- SendGrid API キー、Event Webhook 公開鍵
- Stripe シークレットキー(`sk_live_*` `sk_test_*`)、Webhook シークレット
- Google Cloud Vision API キー(Phase 2)
- マネーフォワード / freee の OAuth Client Secret(Phase 2)
- Vercel デプロイトークン、GitHub Personal Access Token
- Sentry DSN(プロダクション)
- データベース接続文字列(全環境)
- JWT 署名鍵、暗号化鍵
- `CRON_SECRET`、Webhook 署名検証用シークレット

### 顧客・取引情報

- 実在の企業名(seed 社・株式会社シードを含む)
- 実在の人名(亥角さん以外、または公開可と確認済みの名前のみ)
- 実在のメールアドレス、電話番号、住所
- 実際の請求金額、振込先口座番号
- 過去の取引履歴、未払いケースの具体内容
- 顧客との Slack / メールのスクショ・コピー

### 営業秘密

- 弁護士監修済みのメール文面テンプレート(完成版)
- 価格戦略の内部資料(原価計算、利益率)
- seed 社とのレベニューシェア条件、契約書
- 競合分析の詳細(個社名・価格・弱点を含むもの)
- ロードマップの「対外非公開」情報(M&A 検討、撤退条件など)

### その他

- `.env.local` `.env.production` などの実環境変数ファイル
- Supabase の seed データに含まれる本物のデータ
- バックアップ SQL ダンプ
- スクリーンショット内の機密情報(URL バー、ID、メアド)
- 内部 Slack の URL、Notion ページの URL(共有設定によっては漏洩)

---

## 2. リポジトリに含めて良いもの

### コード・設定

- アプリケーションコード(本体)
- マイグレーション SQL(本番データは含まない)
- 型定義
- テストコード(モックデータのみ、実データなし)
- CI / CD 設定(シークレットは `${{ secrets.* }}` で参照)
- ESLint / Prettier / TypeScript 設定

### ドキュメント

- アーキテクチャ、データモデル、フロー図
- 設計判断の根拠
- 利用規約のドラフト(公開予定のもの)
- 開発手順、コーディング規約

### 文面テンプレート(配慮あり)

- **構造・トーンの骨格**は OK
- ただし「弁護士監修済みの完成版」は別管理を推奨(後述)

---

## 3. シークレット管理の実装ルール

### 環境変数の三層構造

```
.env.example      ← Git 管理(変数名のみ、値は空)
.env.local        ← 各開発者のローカルにのみ存在、.gitignore 必須
Vercel / GitHub Secrets ← 本番・CI 環境変数
```

### `.env.example` の書き方

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=      # ← サーバーのみ、絶対に NEXT_PUBLIC_ にしない

# SendGrid
SENDGRID_API_KEY=
SENDGRID_FROM_EMAIL=
SENDGRID_WEBHOOK_PUBLIC_KEY=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# App
NEXT_PUBLIC_APP_URL=
CRON_SECRET=
```

**例の値も書かない**。`SUPABASE_URL=https://xxxxx.supabase.co` のような
それっぽい値でも、本物と混同される事故が起きます。

### `.gitignore` の必須項目

```
# Environment
.env
.env.*
!.env.example

# Local dev
.next/
node_modules/
.vercel/
.turbo/

# Supabase
supabase/.branches/
supabase/.temp/

# Logs
*.log
npm-debug.log*
yarn-debug.log*

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Build
dist/
build/
coverage/

# Local data dumps
*.sql.backup
*.dump
local-data/
```

### GitHub Secrets / Vercel Environment Variables

- **本番**: Vercel Environment Variables(Production)に登録
- **ステージング**: Vercel Environment Variables(Preview)
- **CI**: GitHub Repository Secrets
- **ローカル**: 各開発者の `.env.local`(共有しない、口頭・1Password 等で配布)

### ローテーション方針

- API キー: 6 ヶ月ごと、または流出疑いがあれば即座
- データベースパスワード: 12 ヶ月ごと
- JWT 署名鍵: 12 ヶ月ごと(ローテーション設計が必要)
- 退職者・離脱コントリビューター発生時: 全シークレット即時ローテーション

---

## 4. 流出時の緊急対応

### コミット前に気づいた場合

```bash
# まだステージングのみ
git restore --staged <file>

# まだコミット直後で push 前
git reset --soft HEAD~1
# .env を .gitignore に追加してから再コミット
```

### push 後に気づいた場合

**「履歴から消す」では足りません。** 必ず以下を実施:

1. **すぐにキーをローテーション**(これが最優先、コミット削除より先)
2. 関係サービスにアクセスログがあれば確認(Stripe・Supabase・SendGrid)
3. 影響範囲を Slack で全力AIラボ内に共有
4. `git filter-branch` または `git filter-repo` で履歴から削除
5. GitHub のキャッシュ削除を依頼(`support@github.com`、ただし完全削除は保証されない)
6. fork されている場合は流出済みと見なす

### 専用ツール

- **GitGuardian / TruffleHog**: コミット前のシークレット検出
- **git-secrets**: AWS 等の認証情報パターンを検出
- **GitHub Secret Scanning**: 主要サービスのキーを自動検出

CI に最低 1 つは組み込むこと(推奨: TruffleHog の GitHub Action)。

---

## 5. 営業秘密の扱い

### 弁護士監修済みメール文面

**推奨: Public リポジトリには「骨格テンプレート」のみ置く**

- リポジトリ内: `templates/skeleton/` に変数化された骨組み
- 実運用文面: Supabase の `templates` テーブルに格納、コードからは ID 参照
- 本番投入時に環境別 seed スクリプトで流し込む

```typescript
// OK: 構造のみ
const skeleton = {
  s1: { subject: '...', body: '<<HEADER>><<BODY>><<FOOTER>>' },
};

// NG: 弁護士監修済みの完成文面をハードコード
const final = `お支払期日は ${due_at} でございます。...`;
```

### 価格・契約・パートナー情報

- README やドキュメントに「月額○○円」「seed 社レベニューシェア○○%」を書かない
- 競合比較は「機能差分」レベルにとどめる、具体的な弱点指摘は内部メモ
- 契約書 PDF・Notion へのリンクは絶対に Public リポジトリに置かない

### 内部用ドキュメント

Public リポジトリと並行して、**Private な内部メモ置き場** を別途用意:

- Notion(全力AIラボのワークスペース)
- Private な GitHub リポジトリ(`invoice-reminder-internal` 等)
- ローカル `~/Documents/tonton-private/`(Git 管理外)

Public リポジトリの `docs/` は「OSS 開発者に見せても問題ない情報」のみに限定。

---

## 6. テスト・開発データ

### 絶対に使ってはいけないデータ

- 実在の取引先のメアド・電話番号
- 実際の請求書 PDF
- 本番 DB のダンプ(マスキングなし)
- seed 社の取引履歴

### 推奨データ

- **架空企業**: `株式会社サンプル建設` `合同会社テスト工務店`
- **架空メアド**: `test@example.com`(example.com は予約済みドメイン)
- **金額**: 1,000 円〜 10,000 円のキリの良い数値
- **電話**: `03-0000-0000` `090-0000-0000`
- **住所**: `東京都千代田区サンプル 1-1-1`
- **人名**: `山田太郎` `鈴木花子`(明らかな架空感)

### Faker ライブラリの活用

```typescript
import { fakerJA } from '@faker-js/faker';

const debtor = {
  company_name: fakerJA.company.name(),
  contact_name: fakerJA.person.fullName(),
  email: fakerJA.internet.email(),  // 自動的に example.com 系
  phone: fakerJA.phone.number(),
};
```

---

## 7. Issue / PR / コミットメッセージのルール

### 書いて良いこと

- 機能の説明、設計判断、技術的な議論
- 一般的なバグの再現手順
- パブリックなライブラリのバージョン情報

### 書いてはいけないこと

- 顧客名(seed 社・株式会社シードを含む)
- 実在の金額・取引内容
- スクショ内の機密情報(URL バー、ID、メアド、トークン)
- 内部 Slack の引用、Notion へのリンク
- 「○○さんから聞いた話だけど」のような関係者言及

### スクショの取り方

- URL バー、ブラウザのタブ、ブックマークが映らないようにトリミング
- 開発者ツールを開いている場合、Network タブのリクエストヘッダを隠す
- ローカル開発のスクショは `localhost:3000` 部分も注意(リポジトリ名が映る)
- 必要に応じてモザイク・黒塗り処理

---

## 8. ライセンスの明示

Public リポジトリには **必ず LICENSE ファイル** を置く。

### 推奨: 「閲覧のみ可、商用利用不可」を明示する独自ライセンス

OSS にする予定がない、かつ Public なのは「透明性のため」「採用候補に見せるため」
であれば、All Rights Reserved を明示:

```
Copyright (c) 2026 合同会社スティズム / 全力AIラボ
All Rights Reserved.

This source code is published for transparency and reference purposes only.
No license is granted to use, copy, modify, merge, publish, distribute,
sublicense, or sell copies of this software.

Commercial use, redistribution, and modification require prior written
permission from the copyright holder.

Contact: ismsn526@gmail.com
```

### 代替案

- **AGPL-3.0**: 改変したら必ず公開する義務、SaaS 競合がフォークしづらい
- **Business Source License (BUSL)**: 一定期間は商用利用制限、後に OSS 化
- **完全独自**: コピーライト + 利用条件を README にも記載

選定は弁護士・税理士と相談して決める(年内目標)。

---

## 9. CI / CD でのシークレット保護

### GitHub Actions

```yaml
# OK: secrets 参照
- run: pnpm test
  env:
    SUPABASE_URL: ${{ secrets.SUPABASE_URL }}

# NG: ログに出してしまう
- run: echo "DB: $DATABASE_URL"
```

GitHub Actions のログは Public リポジトリでは **誰でも閲覧可** なので、
`echo` で環境変数を出力しないこと。

### Vercel ビルドログ

Vercel のビルドログも Public リポジトリでは公開設定がある(プロジェクト設定で確認)。
ビルド時のエラーメッセージに環境変数が含まれないよう注意。

### Dependabot / セキュリティアラート

- Dependabot を有効化、自動 PR で依存更新
- GitHub Security Advisories を購読
- `pnpm audit` を CI に組み込み、critical/high は自動失敗

---

## 10. コントリビューター管理

### Public でも貢献は受け付けないなら明示する

README に明記:

```markdown
## Contributing

This project is currently developed by a single team and we are not accepting
external contributions at this time. Issues and questions are welcome.
```

### 受け付ける場合

- CLA(Contributor License Agreement)を整備
- PR テンプレートに「個人情報・機密情報を含まないこと」を明記
- マージ前にコードレビューでシークレット混入をチェック

---

## 11. Claude Code への指示

Claude Code で本リポジトリを操作する際、**以下を必ず守る**:

- `.env.local` `.env.production` などの実環境変数ファイルを **絶対に新規作成しない**
- 既存の `.env.example` に **値を埋めない**(変数名のみ追加可)
- コード内に API キー・パスワード・トークンを直接書かない
  (`process.env.XXX` または `import { env } from '@/lib/env'` 経由)
- テストデータには架空の名前・メアド・金額を使う
- コミットメッセージ・PR 説明に顧客名や内部関係者の名前を書かない
- スクショ生成・サンプル出力でも、実在の企業名を使わない

ユーザーが誤ってシークレットを貼り付けた場合は、**コードに反映する前に警告**し、
環境変数化を提案する。

---

## 12. 定期チェック項目(月次)

- [ ] `git log --all -p | grep -iE "(api[_-]?key|secret|password|token)"` で履歴スキャン
- [ ] GitHub Security Advisories の通知確認
- [ ] Dependabot の未対応 PR を消化
- [ ] `.gitignore` に新規追加すべきパターンがないか確認
- [ ] Public リポジトリで意図しない情報が公開されていないか目視確認
- [ ] 退職・離脱者のアクセス権を取り消し済みか確認

---

## 緊急連絡先

- 流出疑い検知時: 即座に Slack #incidents で共有 → 亥角さんに連絡
- GitHub Support: `support@github.com`
- 主要サービスのサポート窓口は別途まとめておく(Phase 1 リリース時)
