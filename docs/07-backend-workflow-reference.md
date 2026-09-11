# Backend workflowリファレンス

## 1. 表記

- **内部**: Bubble内からのみ呼び出す設定（`expose: false`）。
- **公開**: Workflow APIから呼び出せる設定（`expose: true`）。
- **Privacy無視**: Privacy Rulesを無視してデータを参照・更新する設定。
- 認証欄が「明示なし」の公開Workflowは、設定および呼び出し元を必ず確認してください。

## 2. PDF・領収書（4件）

### BE-PDF-01 `create_pdf`

- **公開設定**: 公開、GET、ユーザー／管理者認証、Privacy無視
- **入力**: `group_id`、購入者メール
- **処理**: モノ購入者向けの一時ログインリンクを作り、`receipt/{group_id}` をMagic PDFで生成する。
- **結果**: `file_url` と成功可否をAPIレスポンスで返す。PDFサービスからのコールバック先は `pdf-generated_mono`。

### BE-PDF-02 `create_pdf_koto`

- **公開設定**: 公開、GET、認証明示なし、Privacy無視
- **入力**: コト予約ID、予約者メール
- **処理**: 一時ログインリンクを作り、`receipt_koto/{予約ID}` をPDF化する。
- **結果**: URLと成功可否を返し、コールバックは `pdf-generated_koto`。

### BE-PDF-03 `pdf-generated_mono`

- **公開設定**: 公開、POST、認証なし、Privacy無視
- **入力**: Group ID、PDF URL、成功可否、メッセージ
- **処理**: Group IDが一致する全 `content_MonoReserve` の `receipt_url` を更新する。
- **要確認**: 成功可否に関係なくURL更新処理が走る構成。コールバックの真正性検証も確認する。

### BE-PDF-04 `pdf-generated_koto`

- **公開設定**: 公開、POST、認証なし、Privacy無視
- **入力**: 予約ID、PDF URL、成功可否、メッセージ
- **処理**: 一意IDが一致する `Content_KotoReserve` の `receipt_url` を更新する。
- **要確認**: 成功可否と署名・認証の検証条件。

## 3. 購入・予約・金額（7件）

### BE-ORD-01 `cart_checkout`

- **公開設定**: 内部、Privacy無視
- **入力**: 数量、商品、販売枠、購入者、プレゼンター、カート、配送先、送料、金額、Group ID等
- **処理**:
  1. 入金済みの `content_MonoReserve` を作成する。
  2. 販売枠、Content、購入者へ購入記録を追加する。
  3. カートをOrderedへ変更する。
  4. 購入メールを予約する。
  5. 枠在庫または商品在庫が0以下なら `soldout` を予約する。

### BE-ORD-02 `cart_checkout_recursive`

- **公開設定**: 内部、Privacy無視
- **入力**: カート一覧、現在位置、購入者、プレゼンター、配送先、送料、メール、Group ID、送料調整額等
- **処理**: カートの現在項目から購入記録を作成し、関連データとカート状態を更新する。残在庫を判定し、次項目へ再帰する。
- **完了条件**: 最終項目で `when_mono_cart_sells` を呼び、Group ID単位の購入完了メールを送る。

### BE-ORD-03 `create_monobuyer`

- **公開設定**: 公開、認証明示なし、Privacy Rules準拠
- **入力**: 直前のモノ購入記録、購入者プロフィール、入金済みに変更するか
- **処理**:
  - 直前の購入情報を複製して2か月後の購入記録を作成する。
  - 指定時は予約状態を入金済みにする。
  - 商品、購入者、定期購入契約へ履歴を追加し、購入メールを送る。
  - キャンセル済みでなければ自分自身をさらに2か月後へ予約し、スケジュールIDを保存する。
- **用途**: モノの定期購入。
- **要確認**: 公開認証、周期が業務要件どおり2か月固定か、失敗時の再実行・重複防止。

### BE-ORD-04 `koto_totalprice`

- **公開設定**: 内部
- **入力**: コト予約
- **処理**: 開催枠の価格 × 参加人数を `total_payment` に保存する。

### BE-ORD-05 `mono_totalprice`

- **公開設定**: 内部
- **入力**: モノ購入
- **処理**: 単品は通常価格 × 数量、定期購入は定期価格 × 数量を `total_payment` に保存する。

### BE-ORD-06 `mono_reservation_deadline_status`

- **公開設定**: 内部
- **入力**: モノ販売枠
- **処理**: 受付期限到達時に `deadline_flg` をyesへ変更する。

### BE-ORD-07 `shipping_fee_0`

- **公開設定**: 内部
- **入力**: Content
- **処理**: `送料調整額` を0へ戻す。

## 4. コンテンツ・開催枠生成（11件）

### BE-CNT-01 `content_add_content_koto`

- **入力**: Content_Koto
- **処理**: 開催枠を親Contentの `26_koto_contents` に追加する。

### BE-CNT-02 `create_ContentKoto1`

- **公開設定**: 内部、Privacy無視
- **入力**: コト一時枠一覧、Content、日数、開始日
- **処理**: 一時枠一覧に対して `create_ContentKoto2` をスケジュールする。

### BE-CNT-03 `create_ContentKoto2`

- **入力**: コト一時枠、Content、対象日
- **処理**: 曜日、除外日、期間条件を満たす場合に `Content_Koto` を作成し、親Contentへ追加する。価格、定員、開催時刻、申込期限、デモ区分を設定する。

### BE-CNT-04 `create_ContentKoto_oneday`

- **入力**: コト一時枠、Content、開催日
- **処理**: 単日開催用の `Content_Koto` を作成し、親Contentへ追加する。

### BE-CNT-05 `create_contentkoto3`

- **入力**: Content_Koto、Content
- **処理**: 開催枠を親Contentへ追加する補助処理。

### BE-CNT-06 `create_content_koto_temp1`

- **公開設定**: 内部、Privacy無視
- **入力**: 一時枠一覧、現在位置、日数、開始日、Content、除外日、曜日、締切設定
- **処理**: 対象日をContentへ記録し、`create_content_koto_temp2` と次の日の自分自身をスケジュールする。

### BE-CNT-07 `create_content_koto_temp2`

- **公開設定**: 内部、Privacy無視
- **入力**: 一時枠一覧、現在位置、日付、Content、除外日、曜日、締切設定
- **処理**: 条件に合うコト開催枠を1件作成して親Contentへ追加し、次の一時枠へ再帰する。

### BE-CNT-08 `create_content_koto_temp_oneday`

- **公開設定**: 内部、Privacy無視
- **入力**: 一時枠一覧、現在位置、単日開催日、Content、締切設定
- **処理**: 一時枠ごとに単日のコト開催枠を作り、次項目へ再帰する。

### BE-CNT-09 `create_content_mono_temp1`

- **公開設定**: 内部、Privacy無視
- **入力**: モノ一時枠一覧、現在位置、日数、開始日、Content、除外日、曜日、締切設定
- **処理**: 対象日をContentへ記録し、`create_content_mono_temp2` と次の日の自分自身をスケジュールする。

### BE-CNT-10 `create_content_mono_temp2`

- **公開設定**: 内部、Privacy無視
- **入力**: 一時枠一覧、現在位置、日付、Content、除外日、曜日、締切設定
- **処理**: 条件に合う `content_Mono` を作成して親Contentへ追加し、期限到達時の `mono_reservation_deadline_status` と次項目をスケジュールする。

### BE-CNT-11 `create_content_mono_temp_oneday`

- **公開設定**: 内部、Privacy無視
- **入力**: 一時枠一覧、現在位置、単日の日付、Content、締切設定
- **処理**: 一時枠ごとに単日のモノ販売枠を作成し、期限処理をスケジュールする。

## 5. 認証・ユーザー補助（3件）

### BE-USR-01 `delete_SignupCode`

- **公開設定**: 内部、Privacy Rulesを無視しない
- **入力**: 認証コード履歴
- **処理**: 指定された `user_SignupCodeHistory` を削除する。

### BE-USR-02 `change_testuser_status`

- **公開設定**: 公開、ユーザー／管理者認証、Privacy無視
- **入力**: User
- **処理**: `test_user` をyesへ変更する。
- **要確認**: 通常ユーザーが他ユーザーを指定できない所有権検証。

### BE-USR-03 `create_demo_user`

- **公開設定**: 内部
- **入力**: 連番
- **処理**: デモ用プロフィールと `demo_user=yes` のUserを作成する。
- **要確認**: 固定形式の初期パスワードを使用しているため、Liveでの利用禁止または強制変更・無効化が必要。

## 6. メール送信（13件）

すべて内部Workflowです。メール送信はSendGridプラグインを利用します。

| ID | Workflow | 入力 | 主な宛先・内容 |
| --- | --- | --- | --- |
| BE-MAIL-01 | `authentication-code` | 宛先、コード、マイページ起点か | 認証コード。起点により本文を切替 |
| BE-MAIL-02 | `contact` | 件名、本文、送信者、宛先、マーケット | 利用者向け控えと運営側通知 |
| BE-MAIL-03 | `create_login_url` | 宛先 | プレゼンター等のログインURL。デモ・所属マーケットを考慮 |
| BE-MAIL-04 | `create_marketowner_login_url` | 宛先 | マーケットオーナー向けログインURL |
| BE-MAIL-05 | `reset_password` | 宛先、トークン、マイページ起点か | パスワード再設定URL |
| BE-MAIL-06 | `when_koto_reserve` | コト予約、宛先、QRコード | 予約者とプレゼンターへ予約内容・参加者・支払方法を通知 |
| BE-MAIL-07 | `when_mono_sells` | モノ購入、宛先 | 購入者、プレゼンター等へ商品、数量、金額、配送先を通知。条件により最大4通 |
| BE-MAIL-08 | `when_mono_cart_sells` | モノ購入一覧、宛先 | カート購入をGroup単位で購入者と運営側へ通知 |
| BE-MAIL-09 | `koto_cancel` | コト予約、宛先 | 予約者とプレゼンターへキャンセル内容を通知 |
| BE-MAIL-10 | `mono_cancel` | モノ購入、宛先 | 購入者とプレゼンターへ単品購入キャンセルを通知 |
| BE-MAIL-11 | `mono_cancel_subscription` | モノ購入、宛先 | 定期購入キャンセルを購入者とプレゼンターへ通知 |
| BE-MAIL-12 | `mono_status_change` | 配送会社、追跡コード、購入者、プレゼンター、購入記録 | 発送・配送状態と追跡情報を購入者へ通知 |
| BE-MAIL-13 | `soldout` | 宛先、プレゼンター、Content、任意の販売枠 | 商品または販売枠を売切れに更新し、プレゼンターへ通知 |

## 7. Stripe Webhook・接続状態（3件）

### BE-STRIPE-01 `stripe_bank_changed`

- **公開設定**: 公開、ユーザー／管理者認証、Privacy無視、未実行でも200応答
- **入力**: Stripeの外部口座イベント
- **処理**: Stripe Account IDに一致する `stripe` を検索し、銀行名、口座末尾、接続日時、通貨関連情報等を更新する。
- **要確認**: Stripe Webhook署名検証とBubble側認証方式の整合。

### BE-STRIPE-02 `stripe_payment_error`

- **公開設定**: 公開、認証なし、Privacy無視
- **入力**: Stripeの定期購入イベント
- **処理**:
  1. Stripe定期購入をキャンセルする。
  2. `subscription_mono` を停止状態にし、スケジュールIDを空にする。
  3. 最新の購入記録をキャンセル済みにする。
  4. 購入者と管理者へ決済失敗メールを送る。
- **要確認**: Workflow名と受信するStripeイベント種別の対応、署名検証、同一イベント再送時の冪等性。

### BE-STRIPE-03 `srtipe_deleted`

- **公開設定**: 公開、認証なし、Privacy無視
- **処理**: 現在はアクションが設定されていない受信用Workflow。
- **要確認**: 綴りを含む名称、利用中か、削除イベントで必要な後処理。

## 8. 管理・データ移行（4件）

### BE-ADM-01 `20260630_content_method_change`

- **公開設定**: 内部
- **入力**: Content（ただし実処理は検索結果全体）
- **処理**: 販売方法が空の全Contentを「配送販売」へ変更する。
- **用途**: 既存データ移行。

### BE-ADM-02 `add_firstmarket`

- **公開設定**: 内部
- **入力**: User
- **処理**: 現在の `belongs_market` を `first_market` にコピーする。

### BE-ADM-03 `add_group_id_monoreserve`

- **公開設定**: 内部、Privacy無視
- **入力**: モノ購入一覧、開始Group ID
- **処理**: Group IDが空の購入記録へ連番を付け、残りがあれば自分自身を再実行する。
- **用途**: 既存データ移行。

### BE-ADM-04 `add_taxrate`

- **公開設定**: 内部
- **入力**: Content
- **条件**: Contentの税率が空。
- **処理**: Contentと関連する全モノ購入記録の税率を消費税10%へ設定する。

## 9. 公開Workflow一覧と優先確認

| Workflow | 認証 | Privacy無視 | 優先確認事項 |
| --- | --- | --- | --- |
| `create_pdf` | ユーザー／管理者 | yes | メールとGroup IDの所有権 |
| `create_pdf_koto` | 明示なし | yes | 予約IDとメールの所有権 |
| `pdf-generated_mono` | なし | yes | コールバック署名、Group ID改ざん |
| `pdf-generated_koto` | なし | yes | コールバック署名、予約ID改ざん |
| `change_testuser_status` | ユーザー／管理者 | yes | 他ユーザー変更の防止 |
| `create_monobuyer` | 明示なし | no | 外部実行防止、重複作成防止 |
| `stripe_bank_changed` | ユーザー／管理者 | yes | Stripe署名検証 |
| `stripe_payment_error` | なし | yes | Stripe署名、イベント種別、冪等性 |
| `srtipe_deleted` | なし | yes | 未使用なら閉鎖 |

## 10. Backend workflow変更時の確認項目

- 入力レコードが現在ユーザーまたは正当な外部サービスに属するか。
- Privacy Rulesを無視する必要が本当にあるか。
- 再帰の終了条件と最大件数が明確か。
- 同一リクエストの再送で重複作成・二重メール・二重返金が起きないか。
- 途中失敗後に再開できるか、または安全にやり直せるか。
- Test／Live、デモ／実、マーケット境界が保たれるか。
- スケジュールIDを保存し、キャンセルや変更時に古い予定を停止しているか。

