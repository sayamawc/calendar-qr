# カレンダー登録QRコード生成アプリ 仕様書

## プロジェクト概要

イベント情報を入力するとQRコードを生成するWebアプリ。
QRコードをスマートフォンで読み取ると**カレンダー選択ページ**が開き、Google カレンダー・Apple カレンダー・Outlook 等にワンタップで予定を追加できる。

---

## 技術スタック

- HTML / CSS / JavaScript（単一ファイル構成）
- QRコード生成ライブラリ：`qrcode.js`（CDN: `https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js`）
- 外部サーバー・バックエンド不要
- 公開先：GitHub Pages 等の静的ホスティング（QRコード読み取り時にURLアクセスが必要なため）

---

## ファイル構成

```
calendar-qr/
└── index.html   ← 1ファイルですべて完結（生成画面・登録画面を兼用）
```

---

## 画面構成

`index.html` はURLのクエリパラメータの有無で2つのモードを切り替える。

### モード判定

| 条件 | 表示画面 |
|---|---|
| クエリパラメータなし | 生成画面（イベント入力フォーム） |
| `?title=...&start=...&end=...` あり | 登録画面（カレンダー選択） |

---

### 生成画面（パラメータなし）

#### 入力フォーム

| フィールド | 種別 | 必須 | 備考 |
|---|---|---|---|
| タイトル | テキスト入力 | ✅ | プレースホルダー例：「会議」「打ち合わせ」 |
| 開始日時 | datetime-local | ✅ | |
| 終了日時 | datetime-local | ✅ | 開始日時より後であることを検証 |
| 場所 | テキスト入力 | ❌ | 任意入力 |
| 説明・メモ | テキストエリア | ❌ | 任意入力 |

#### ボタン

- **QRコードを生成** ボタン（メインアクション）
- **.icsファイルをダウンロード** ボタン（QRコード生成後に表示）

#### QRコード表示エリア

- 生成ボタン押下後に表示
- QRコード画像（256×256px 目安）
- 「このQRコードをスマートフォンで読み取ると、カレンダーに予定を追加できます」の説明テキスト

---

### 登録画面（パラメータあり）

QRコード読み取り後にスマートフォンで表示される画面。

#### イベント概要表示

クエリパラメータから取得した以下の情報を表示する。

- タイトル
- 開始日時・終了日時（ローカル時刻に変換して表示）
- 場所（入力がある場合）
- 説明（入力がある場合）

#### カレンダー追加ボタン

- **Google カレンダーに追加** ボタン — Google Calendar URL を開く
- **Apple カレンダー / Outlook に追加** ボタン — .ics ファイルをダウンロード

---

## 機能仕様

### クエリパラメータ形式

QRコードにエンコードするURLのパラメータ：

| パラメータ | 必須 | 値の形式 | 例 |
|---|---|---|---|
| `title` | ✅ | 文字列（encodeURIComponent済み） | `%E4%BC%9A%E8%AD%B0` |
| `start` | ✅ | `YYYYMMDDTHHmmssZ`（UTC） | `20260307T010000Z` |
| `end` | ✅ | `YYYYMMDDTHHmmssZ`（UTC） | `20260307T020000Z` |
| `location` | ❌ | 文字列（encodeURIComponent済み） | `%E4%BC%9A%E8%AD%B0%E5%AE%A4A` |
| `description` | ❌ | 文字列（encodeURIComponent済み） | `%E8%AD%B0%E9%A1%8C...` |

### QRコード生成

- 自サイトURL + 上記クエリパラメータをQRコードにエンコードする
- ベースURL は `location.origin + location.pathname` から自動取得する
- `qrcode.js` の `QRCode` クラスを使用
- エラー訂正レベル：`M`
- サイズ：256×256px

### Google Calendar URL 生成（登録画面）

クエリパラメータから以下の形式のURLを生成してリンクする。

```
https://calendar.google.com/calendar/render?action=TEMPLATE
  &text={タイトル}
  &dates={開始}%2F{終了}
  &location={場所}（入力がある場合のみ）
  &details={説明}（入力がある場合のみ）
```

### iCalendar（.ics）データ生成

入力値をもとに以下の形式の文字列を生成する。生成画面の .ics ダウンロードと、登録画面の Apple カレンダー / Outlook ボタンで共用する。

```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//Calendar QR Generator//JP
BEGIN:VEVENT
UID:{タイムスタンプ}@calendar-qr
DTSTAMP:{現在日時 UTC, YYYYMMDDTHHmmssZ形式}
DTSTART:{開始日時 UTC, YYYYMMDDTHHmmssZ形式}
DTEND:{終了日時 UTC, YYYYMMDDTHHmmssZ形式}
SUMMARY:{タイトル}
LOCATION:{場所}（入力がある場合のみ）
DESCRIPTION:{説明}（入力がある場合のみ）
END:VEVENT
END:VCALENDAR
```

**注意：** 日時はブラウザのローカル時刻をUTCに変換して出力する。

### .icsファイルダウンロード

- iCalendar文字列を `text/calendar` のBlobとして生成
- `<a download="event.ics">` を動的生成してクリックする方式で実装

### バリデーション（生成画面のみ）

- タイトル未入力時：アラートを表示して処理中断
- 開始日時・終了日時未入力時：アラートを表示して処理中断
- 終了日時 ≤ 開始日時の場合：「終了日時は開始日時より後に設定してください」を表示して処理中断

---

## UI・スタイル指針

- シンプルかつ清潔感のあるデザイン
- モバイルフレンドリー（スマートフォンでの入力・確認を考慮）
- フォントサイズ：入力欄 16px 以上（iOS のズーム防止）
- 最大幅 480px 程度でセンタリング
- 配色：白背景 + アクセントカラー1色（青系を推奨）
- QRコード生成後はフォームとQRコードが同一画面に収まるよう配慮
- 登録画面はスマートフォンでの操作を前提としたシンプルなレイアウト

---

## 動作確認ポイント

1. PC のブラウザで入力→QRコード生成されること
2. QRコードを読み取ると登録画面が表示され、イベント概要が正しいこと
3. 「Google カレンダーに追加」→ Google Calendar の予定作成画面が開くこと（Android / iOS）
4. 「Apple カレンダー / Outlook に追加」→ .ics ダウンロード → カレンダーアプリに取り込めること（iOS / PC）
5. 生成画面の .ics ダウンロードボタンから直接カレンダーに取り込めること
6. バリデーションが正しく動作すること

---

## 実装上の補足

- `qrcode.js` は `new QRCode(element, { text, width, height, correctLevel })` で使用する
- 再生成時は `document.getElementById('qrcode').innerHTML = ''` で前のQRコードをクリアしてから再生成する
- URLのクエリパラメータは `URLSearchParams` で取得する
- 登録画面の日時表示は `toLocaleString('ja-JP')` でローカル時刻に変換する
- iCalendar の改行は `\r\n`（CRLF）を使用する（RFC 5545 準拠）
- 長い行（75オクテット超）の折り返し（line folding）は今回は省略可

---

## スコープ外（今回は対象外）

- 繰り返し予定（RRULE）
- 参加者・招待（ATTENDEE）
- アラーム・リマインダー（VALARM）
- サーバーサイド処理
- データの保存・履歴機能
