# 外部連携・セキュリティ・運用

## 1. 外部連携

| サービス・機能 | 用途 | 実装の要点 |
| --- | --- | --- |
| Stripe Connect | プレゼンター・マーケットオーナーの決済受取口座 | アカウント作成、オンボーディングリンク、ログインリンク、口座状態取得・削除 |
| Stripe Checkout | 単品・カート・定期購入のカード決済 | 商品金額、数量、手数料、送金先、成功・キャンセルURL、取引識別情報を渡す |
| SendGrid | トランザクションメール | 認証、ログイン、購入、予約、キャンセル、発送、問い合わせ等 |
| Magic PDF | 領収書生成 | 専用ページと一時ログインリンクからPDFを生成 |
| HeartRails Geo API | 郵便番号検索 | 郵便番号から住所候補を取得 |
| QRコード | 購入・予約受付 | QR生成、画像保存、カメラ読取、取引状態更新 |
| Google Analytics | アクセス解析 | 全体のheadにGoogle tagを設定 |

外部連携の秘密値はBubbleのPrivate Keyまたはプラグイン設定で管理し、GitHub文書には保存しません。

## 2. API設定

- Workflow API: 有効
- Data API: 無効
- Swaggerドキュメント: 非表示
- 再帰Workflow上限: Test／Liveともに5000
- PDF生成およびStripe関連Webhook等、一部のBackend workflowは外部公開されています。
- 外部公開WorkflowのうちPrivacy Rulesを無視するものがあるため、認証方式、推測困難な識別子、署名検証、パラメータ検証を個別に確認する必要があります。

## 3. アプリ全体設定

| 項目 | 現行値 |
| --- | --- |
| 主言語 | 日本語 |
| iframe | 拒否 |
| Cookie opt-in必須 | 無効 |
| 新規データタイプを既定で非公開 | 無効 |
| Search Privacy Rules強制 | automatic |
| Sitemap | 無効 |
| Robots設定 | 無効 |
| Canonical URL | 無効 |
| タイムゾーン制御 | 無効 |

SEOの共通タイトル・説明とソーシャル共有画像が設定されています。マーケットページでは `market` のSEOタイトル、説明、メイン画像を使用します。

## 4. 導入プラグイン

主な導入プラグインは次のとおりです。

- Stripe.js 2、SendGrid、Magic PDF
- Colour QR Code Generator、QR Code Scanner
- Calendar tool、Fuzzy search & Autocomplete
- Local Storage & Cookies、Floppy（localStorage / List Shifter）
- File Downloader、CSV Download Converter（Shift JIS）
- Reveal & Hide Password、Screenshot the Group、Toolbox

プラグイン更新時は、権限、互換性、個人情報送信先、Test環境での回帰動作を確認してください。

## 5. セキュリティ上の要確認事項

優先度順の確認項目です。これは脆弱性の確定ではなく、現行設定から確認が必要と判断した箇所です。

1. **予約・購入データの公開範囲**  
   `content_MonoReserve` と `Content_KotoReserve` は全項目を誰でも閲覧・検索できるPrivacy Ruleです。住所、電話、購入者、決済識別情報を含むため、必要最小限の本人・出品者・運営者アクセスへ絞ることを推奨します。

2. **Privacy Rules未設定のデータタイプ**  
   `UserInformation`、カート、問い合わせ、通知、定期購入等について、ロールごとの参照・検索・更新範囲を定義する必要があります。

3. **公開Backend workflow**  
   PDF生成、定期購入記録作成、Stripe関連処理等の公開エンドポイントについて、認証、Webhook署名、所有権、入力値、リプレイ耐性を確認してください。

4. **デモ・実データ分離**  
   一部データは `demo_user` / `is demo` で分離されていますが、全関連データで一貫しているか確認が必要です。

5. **テスト・管理用機能**  
   `99_test_function` やデータ移行用Backend workflowがLiveで一般利用者から実行できないことを確認してください。

6. **Cookie同意・分析タグ**  
   Google Analyticsが設定され、Cookie opt-in必須設定は無効です。適用法令とプライバシーポリシーに沿って同意管理を確認してください。

## 6. 運用チェックリスト

### リリース前

- BubbleのTestとLiveの差分を確認する
- 一般ユーザー、プレゼンター、マーケットオーナー、管理者で主要導線を確認する
- 未ログイン状態で管理URLを直入力してもアクセスできないことを確認する
- デモユーザーから実データ、通常ユーザーからデモデータが見えないことを確認する
- Stripeの成功、キャンセル、エラー、Webhook再送を確認する
- 在庫・定員の境界値と同時購入を確認する
- メールの宛先、本文、リンク先をTest環境で確認する
- Privacy Rulesの検査を実施する

### 定期運用

- Stripe接続アカウントの決済・入金有効状態を監視する
- 失敗したBackend workflowとメール送信をログで確認する
- 売切れ、期限切れ、定期購入のスケジュール処理を確認する
- 不要な認証コード履歴、テストデータ、生成PDFの保持期間を確認する
- プラグインおよびAPI仕様変更を確認する

## 7. 推奨テスト観点

| 領域 | 最低限のケース |
| --- | --- |
| 認証 | 新規登録、誤コード、期限切れ、既存メール、パスワード再設定、退会済み |
| 権限 | 4ロール×ログイン有無×URL直入力 |
| モノ | 配送／現地、単品／カート／定期、在庫1・0、売切れ、キャンセル |
| コト | 現金／カード、定員境界、期限切れ、複数参加者、キャンセルポリシー |
| 決済 | 成功、キャンセル、失敗、二重送信、Webhook再送、0円相当 |
| 配送 | 地域別／一律／無料、離島、追跡情報、発送済み |
| PDF | モノのGroup ID、コト予約、認証切れ、生成失敗 |
| データ分離 | Test／Live、デモ／実、異なるマーケット |

## 8. 変更管理

Bubble変更時は、変更理由、対象ロール、影響ページ、データ変更、移行要否、外部連携影響、ロールバック方法をPull Requestに記載します。Privacy Rules、決済、予約・購入状態、公開Backend workflowの変更はレビュー必須とする運用を推奨します。

