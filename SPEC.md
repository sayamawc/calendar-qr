# カレンダー登録QRコード生成アプリ 仕様書

## プロジェクト概要

イベント情報を入力するとiCalendar形式のQRコードを生成するWebアプリ。
QRコードをスマートフォンで読み取るとカレンダーアプリ（Google カレンダー・Apple カレンダー・Outlook等）に予定を追加できる。

---

## 技術スタック

- HTML / CSS / JavaScript（単一ファイル構成）
- QRコード生成ライブラリ：`qrcode.js`（CDN: `https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js`）
- 外部サーバー・バックエンド不要

---

## ファイル構成

```
calendar-qr/
└── index.html   ← 1ファイルですべて完結
```

---

## 画面構成

### 入力フォーム

| フィールド | 種別 | 必須 | 備考 |
|---|---|---|---|
| タイトル | テキスト入力 | ✅ | プレースホルダー例：「会議」「打ち合わせ」 |
| 開始日時 | datetime-local | ✅ | |
| 終了日時 | datetime-local | ✅ | 開始日時より後であることを検証 |
| 場所 | テキスト入力 | ❌ | 任意入力 |
| 説明・メモ | テキストエリア | ❌ | 任意入力 |

### ボタン

- **QRコードを生成** ボタン（メインアクション）
- **.icsファイルをダウンロード** ボタン（QRコード生成後に表示）

### QRコード表示エリア

- 生成ボタン押下後に表示
- QRコード画像（256×256px 目安）
- 「このQRコードをスマートフォンで読み取ると、カレンダーに予定を追加できます」の説明テキスト

---

## 機能仕様

### iCalendar（.ics）データ生成

入力値をもとに以下の形式の文字列を生成する。

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

### QRコード生成

- **Google Calendar URL** をQRコードにエンコードする
  - 生のiCalendarテキストはスマートフォンのQRリーダーでカレンダーデータとして認識されないため、Google Calendar URLを使用する
- URL形式：`https://calendar.google.com/calendar/render?action=TEMPLATE&text={タイトル}&dates={開始}%2F{終了}&location={場所}&details={説明}`
  - `dates` の値は `YYYYMMDDTHHmmssZ/YYYYMMDDTHHmmssZ`（UTC）
  - `location`, `details` は入力がある場合のみパラメータに含める
  - 各パラメータ値は `encodeURIComponent` でエンコードする
- `qrcode.js` の `QRCode` クラスを使用
- エラー訂正レベル：`M`
- サイズ：256×256px

### .icsファイルダウンロード

- iCalendar文字列を `text/calendar` のBlobとして生成
- `<a download="event.ics">` を動的生成してクリックする方式で実装

### バリデーション

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

---

## 動作確認ポイント

1. PC のブラウザで入力→QRコード生成されること
2. スマートフォンのカメラでQRコードを読み取ると、カレンダーアプリへの追加確認ダイアログが表示されること（iOS / Android）
3. .icsダウンロードボタンから直接カレンダーに取り込めること
4. バリデーションが正しく動作すること

---

## 実装上の補足

- `qrcode.js` は `new QRCode(element, { text, width, height, correctLevel })` で使用する
- 再生成時は `document.getElementById('qrcode').innerHTML = ''` で前のQRコードをクリアしてから再生成する
- QRコードには Google Calendar URL をエンコードする（iCalendar テキストはデータ量が多く、またスマートフォンのQRリーダーがカレンダーデータとして認識しないため）
- .ics ダウンロード用の iCalendar 文字列は従来通り生成する（改行は `\r\n`（CRLF）、RFC 5545 準拠）
- 長い行（75オクテット超）の折り返し（line folding）は今回は省略可

---

## スコープ外（今回は対象外）

- 繰り返し予定（RRULE）
- 参加者・招待（ATTENDEE）
- アラーム・リマインダー（VALARM）
- サーバーサイド処理
- データの保存・履歴機能
