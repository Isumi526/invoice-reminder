# 09. 用語集

プロジェクト内で使う用語の統一。コード上の命名・UI 表記・ドキュメント記述すべてで使い分ける。

## ドメイン用語

### 請求関連

| 用語 | コード上の表現 | UI 表記 | 意味 |
|---|---|---|---|
| 請求書 | `Invoice` `invoice` | 請求書 | 発行された請求文書 1 件 |
| 請求書番号 | `invoice_number` | 請求書番号 | 発行者がつける一意の識別子 |
| 請求額 | `amount` | ご請求額 | 税抜の請求金額 |
| 税額 | `tax_amount` | 消費税 | 消費税額 |
| 合計請求額 | `total_amount` | ご請求総額 | 税込合計 |
| 発行日 | `issued_at` | 請求書発行日 | 請求書を発行した日 |
| 支払期日 | `due_at` | お支払期日 | 入金されるべき期日 |
| 入金日 | `paid_at` | 入金日 | 実際に入金された日 |
| 経過日数 | `overdue_days` | 経過日数 | 期日からの経過日数(マイナスは期日前) |
| 部分入金 | `partial_paid_amount` | 一部入金 | 全額に満たない入金 |

### 取引先関連

| 用語 | コード上の表現 | UI 表記 | 意味 |
|---|---|---|---|
| 取引先 | `Debtor` `debtor` | 取引先 | 請求の宛先となる企業・個人 |
| 担当者 | `contact_name` | ご担当者様 | 取引先の担当者 |
| 取引停止 | `is_blocked` | 取引停止中 | この取引先への送信停止フラグ |

注: コード上は債権者視点で `debtor`(債務者)を使うが、**UI 上では常に「取引先」と表記**。
理由: ユーザー(債権者)の主観として、催促前から「債務者」と呼ぶのは不自然。

### リマインド関連

| 用語 | コード上の表現 | UI 表記 | 意味 |
|---|---|---|---|
| リマインド | `Reminder` `reminder` | お振込のご確認 | 送信される催促メール 1 通 |
| ステージ | `stage` | 段階 | S1〜S5 のレベル |
| 送信予定 | `scheduled` | 送信予定 | まだ送られていない |
| 送信済み | `sent` | 送信済み | 送信完了 |
| バウンス | `bounced` | 不達 | メールが届かなかった |
| 開封 | `opened` | 開封確認 | トラッキングピクセルが反応した(精度限界あり) |
| クリック | `clicked` | リンククリック | 本文中のリンクが押された |

### 金額関連

| 用語 | コード上の表現 | UI 表記 | 意味 |
|---|---|---|---|
| 遅延損害金 | `late_fee` | 遅延損害金 | 期日超過に伴う追加請求 |
| 利率 | `late_fee_rate` | 利率 | 年利(decimal、例: 0.030) |
| 元本 | `principal` | 元本 | 遅延損害金を除く請求額 |

### 組織・ユーザー

| 用語 | コード上の表現 | UI 表記 | 意味 |
|---|---|---|---|
| 組織 | `Organization` `organization` | 会社 / 事業者 | マルチテナント単位の企業 |
| メンバーシップ | `Membership` `membership` | メンバー登録 | ユーザーと組織の所属関係 |
| プロフィール | `Profile` `profile` | プロフィール | ユーザー個人の情報 |
| ロール | `role` | 権限 | owner / admin / member |

## 状態の用語

### 請求書ステータス

| コード値 | UI 表記 | 説明 |
|---|---|---|
| `draft` | 下書き | 未確定 |
| `active` | 督促中 / 期日前 | リマインド対象 |
| `paid` | 入金確認済み | 完了 |
| `cancelled` | 取消 | 請求自体を取消 |
| `escalated` | 専門家相談中 | S5 到達後の手動オペレーション中 |

注: UI 上では `active` を「督促中」「期日前」「期限間近」などコンテキストで使い分け。

### リマインドステータス

| コード値 | UI 表記 | 説明 |
|---|---|---|
| `scheduled` | 送信予定 | まだキューに入っていない |
| `queued` | 送信中 | 送信処理中 |
| `sent` | 送信済み | 送信完了 |
| `failed` | 送信失敗 | 一時的な失敗 |
| `bounced` | 不達 | メールアドレスが無効 |
| `opened` | 開封 | トラッキング反応(参考値) |
| `clicked` | クリック | リンク押下 |
| `cancelled` | キャンセル | 入金確認等で停止 |

## 法務・規約用語

| 用語 | 説明 |
|---|---|
| 弁護士法 72 条 | 非弁護士による法律事務の禁止 |
| 非弁行為 | 弁護士でない者が報酬目的で法律事務を業として行うこと |
| 債権回収業 | 法務大臣の許可が必要、特定の債権のみ取り扱い可 |
| 内容証明郵便 | 郵便局が文面を証明する書留 |
| 少額訴訟 | 60 万円以下の金銭請求を簡易裁判所で行う手続 |
| 商事法定利率 | 2020 年改正後は民事と統一で年 3% |
| 特定電子メール法 | 広告メールの規制法、督促メールは対象外だが配信停止等の配慮は推奨 |
| 電子帳簿保存法 | 電子取引データの保存義務(2024 年義務化) |
| インボイス制度 | 適格請求書等保存方式(2023 年開始) |

## 技術用語

| 用語 | 説明 |
|---|---|
| RLS | Row Level Security、Supabase の行レベルアクセス制御 |
| Server Action | Next.js App Router のサーバー側関数 |
| Edge Function | Supabase の Deno ランタイム関数 |
| Webhook | サービス間の HTTP プッシュ通知 |
| Magic Link | パスワードレス認証のリンク方式 |
| SPF/DKIM/DMARC | メール送信ドメイン認証 |
| TTFB | Time to First Byte |
| LCP | Largest Contentful Paint |
| MRR | Monthly Recurring Revenue 月次経常収益 |

## 命名規約

### コード上の英語表記

- **テーブル名**: 複数形・スネークケース(`invoices`, `audit_logs`)
- **カラム名**: スネークケース、ブール値は `is_` `has_` プレフィックス(`is_blocked`)
- **TypeScript 型**: PascalCase(`Invoice`, `Reminder`)
- **関数名**: camelCase 動詞始まり(`createInvoice`, `markAsPaid`)
- **enum 値**: スネークケース小文字(`paid`, `partial_paid`)

### UI 上の日本語表記

- **動詞は丁寧語**: 「送信する」「確認する」「削除する」
- **金額表記**: `¥330,000` 形式、3 桁区切り、税込前提
- **日付表記**: `2026/04/30(火)` 形式、曜日付き
- **時刻表記**: `09:00` 形式(24 時間)
- **経過日数**: `超過 22 日` 形式(マイナスは「あと N 日」)

## 略語

| 略語 | 正式 |
|---|---|
| MVP | Minimum Viable Product |
| LP | Landing Page |
| KPI | Key Performance Indicator |
| OCR | Optical Character Recognition |
| SaaS | Software as a Service |
| API | Application Programming Interface |
| UI/UX | User Interface / User Experience |
| RSC | React Server Components |
| SLA | Service Level Agreement |
| PII | Personally Identifiable Information |

## ステークホルダー

| 表記 | 説明 |
|---|---|
| Tonton | 本プロダクトのブランド名(顧客向け呼称) |
| invoice-reminder | 本プロダクトの技術的識別子(GitHub・Vercel・Supabase 等の内部名) |
| 亥角さん / 開発主 | 亥角貴治、合同会社スティズム代表、全力AIラボ運営、本プロジェクトの開発主 |
| seed 社 / β顧客 | 株式会社シード、本プロダクトの企画相談元かつ初期顧客 |
| sido | 別プロダクト(施工台帳 SaaS)、seed 社向けに別途開発中。Tonton とは独立 |
| ユーザー | Tonton を契約して使う事業者(債権者) |
| 取引先 | ユーザーの請求宛先(債務者)、Tonton を直接使うわけではない |

### ブランド名(Tonton)と技術名(invoice-reminder)の使い分け

| 文脈 | 使う表記 |
|---|---|
| LP・マーケティング | Tonton |
| 契約書・利用規約 | Tonton(運営: 合同会社スティズム) |
| メール文面の差出人 | ユーザー組織名(Tonton は出さない) |
| UI 上のロゴ・ヘッダー | Tonton |
| GitHub リポジトリ | invoice-reminder |
| Vercel プロジェクト | invoice-reminder |
| Supabase プロジェクト | invoice-reminder |
| `package.json` の `name` | invoice-reminder |
| Docker イメージ名 | invoice-reminder |
| 環境変数 | プロダクト名を含めない(`SUPABASE_URL` 等) |
| 内部 Slack チャンネル | #invoice-reminder または #tonton(チームの好みで統一) |

## 内部メモ用語

以下は **内部資料のみで使う** 表現。顧客・取引先向け資料には絶対に出さない:

- 「催促」「督促」(社内議論で使うが、UI 上ではより柔らかい表現を選ぶ)
- 「踏み倒し」(社内議論用、対外的には絶対に使わない)
- 「ガミガミ」「キツめ」(トーン議論用)
- 「ブラックリスト」(社内分類でも避ける、`is_blocked` で機械的に表現)
