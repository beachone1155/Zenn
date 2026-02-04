---
title: "【GAS】Teamsの「空き」予定をGoogleカレンダー連携時に強制的に「予定あり」に変換する"
emoji: "📝"
type: "tech"
topics: ["gas","google-apps-script","outlook","teams","google-calendar"]
published: true
---


# 【GAS】Teamsの「空き」予定をGoogleカレンダー連携時に強制的に「予定あり」に変換する

会社では Microsoft Teams (Outlook)、プライベートでは Google カレンダーを使用しているエンジニアは多いのではないでしょうか。

Google カレンダーの「予約スケジュール」機能を使って日程調整を自動化しようとした際、一つの大きな壁にぶつかりました。それは、**Teams 側で「空き」として入れている予定が、Google カレンダー側でも「空き」と判定され、予約をブロックしてくれない**という問題です。

今回は、Google Apps Script (GAS) を中継（プロキシ）として利用し、この問題をスマートに解決する方法を紹介します。

## 抱えていた課題

Google カレンダーの予約スケジュール機能は、連携しているカレンダーの「予定あり」の時間帯を自動的にブロックします。しかし、以下のようなケースで不都合が生じます。

1.  **Teams 側の運用**: 社内向けに「今日はこの作業をしています」という指標として終日予定を入れるが、ステータスは「空き」に設定している。
2.  **標準同期の挙動**: Google カレンダーの標準機能（URLで追加）で Teams の ICS を読み込むと、「空き」ステータスまで忠実に同期される。
3.  **結果**: Google 予約スケジュールがその時間を「空き時間」と判定してしまい、本来避けてほしい時間に予約が入ってしまう。

Google カレンダー側には「読み込んだ予定を強制的に『予定あり』にする」設定が存在しません。そこで、GAS をプロキシとして動作させ、ICS ファイルの中身を動的に書き換えてから Google カレンダーに渡す構成を構築しました。

## アーキテクチャ

仕組みは非常にシンプルです。

*   **Before**: Teams (ICS) `TRANSPARENT` → Googleカレンダー `空き`（ブロックされない😭）
*   **After**: Teams (ICS) `TRANSPARENT` → **GAS (書き換え)** → Googleカレンダー `OPAQUE`（ブロックされる😤）

## 手順1：TeamsのICSリンクを取得する

まずは同期元となる Outlook (Web版) の設定から ICS リンクを取得します。

1.  **設定** > **カレンダー** > **共有カレンダー** に移動。
2.  「カレンダーを公開する」セクションで、公開したいカレンダーを選択。
3.  「すべて表示可能」などの権限を選択して「公開」をクリック。
4.  発行された **ICS リンク**（`https://outlook.office365.com/.../calendar.ics`）をコピーしておきます。

## 手順2：GASを作成する

[script.google.com](https://script.google.com/) で新規プロジェクトを作成し、以下のコードを記述します。

やっていることは、ICS のテキストデータを取得し、`TRANSP:TRANSPARENT`（予定なし）を `TRANSP:OPAQUE`（予定あり）に全置換して返却するだけです。

```javascript
// ここにTeamsのICSリンク(https://...)を貼り付けてください
const TEAMS_ICS_URL = 'https://outlook.office365.com/owa/calendar/xxxxxxxx/calendar.ics';

function doGet(e) {
  try {
    // 1. TeamsからICSデータを取得
    const response = UrlFetchApp.fetch(TEAMS_ICS_URL);
    let content = response.getContentText();

    // 2. 「予定なし(TRANSPARENT)」を「予定あり(OPAQUE)」に強制置換
    // これによりGoogleカレンダー側が「ブロックされた予定」として認識する
    content = content.replace(/TRANSP:TRANSPARENT/g, 'TRANSP:OPAQUE');

    // 念のためMicrosoft独自のプロパティも書き換え（Free -> Busy）
    content = content.replace(/X-MICROSOFT-CDO-BUSYSTATUS:FREE/g, 'X-MICROSOFT-CDO-BUSYSTATUS:BUSY');

    // 3. ICSファイルとしてレスポンスを返す
    return ContentService.createTextOutput(content)
      .setMimeType(ContentService.MimeType.ICAL);

  } catch (error) {
    return ContentService.createTextOutput("Error: " + error.toString());
  }
}
```

## 手順3：デプロイ（最重要ポイント）

Google カレンダーという「外部サービス」がこのスクリプトにアクセスできるようにするため、公開設定には注意が必要です。

1.  右上の「**デプロイ**」 > 「**新しいデプロイ**」をクリック。
2.  「種類の選択」から「**ウェブアプリ**」を選択。
3.  設定を以下のように変更：
    *   **次のユーザーとして実行**: `自分`
    *   **アクセスできるユーザー**: `全員` (**ここを間違えるとGoogleカレンダーが読み込めません**)
4.  デプロイを実行し、発行された **ウェブアプリのURL**（`https://script.google.com/macros/s/.../exec`）をコピーします。

## 手順4：Googleカレンダーに登録

最後に、作成した GAS 経由の URL を Google カレンダーに登録します。

1.  Google カレンダーを開き、「**他のカレンダー**」の横にある「**＋**」 > 「**URLで追加**」をクリック。
2.  先ほどコピーした **GAS のウェブアプリ URL** を貼り付けて「カレンダーを追加」をクリック。

これで、Teams 側が「空き」であっても、Google カレンダー上では「予定あり」として登録され、予約機能でもしっかりブロックされるようになります。

## トラブルシューティング：反映されない場合

実装時に遭遇しやすい「Google カレンダーに予定が表示されない」という現象への対策です。

### 1. 認証の壁（Googleの警告画面）
デプロイ直後の URL にブラウザでアクセスしようとすると、「このアプリは Google で確認されていません」という警告画面が出ることがあります。Google カレンダーのボットはこの画面を突破できないため、読み込みエラーになります。
![alt text](https://beachone1155.vercel.app/images/teams-google-calendar-gas-proxy-warning.png)
*   **解決策**: シークレットウィンドウなどで一度その URL にアクセスし、警告が出る場合は「詳細」→「安全ではないページに移動」を押して権限を許可しておきます。

### 2. Googleカレンダーのキャッシュ
Google カレンダーは一度読み込みに失敗すると、同じ URL にはしばらくアクセスしに行かない（キャッシュする）仕様があるようです。

*   **解決策**: 再登録する際、URL の末尾にダミーのパラメータをつけて**「別のURL」として認識させる**とうまくいきます。
    *   例: `.../exec` → `.../exec?v=2`

## おわりに

Teams と Google カレンダーの二重運用において、「予定は入っているのに予約されてしまった」という事故を防ぐには、この GAS プロキシが現状最強の解決策です。

同じ悩みを持つエンジニアの方は、ぜひ試してみてください。
