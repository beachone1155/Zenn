---
title: "GitHub Actions無料枠が尽きても投稿経路を止めない：外部APIとGit連携のフェイルオーバー設計"
emoji: "🛟"
type: "tech"
topics: ["githubactions","automation","ci","zenn"]
published: true
---

# GitHub Actions無料枠が尽きても投稿経路を止めない：外部APIとGit連携のフェイルオーバー設計

## はじめに

複数メディアへ記事を自動配信していると、つい「GitHub Actionsが動く = 配信基盤が動く」と考えてしまいます。

ところが実運用では、Actionsの無料枠を使い切った瞬間に、コードは正常でもrunner自体が起動しなくなることがあります。さらに、ホスティング側のデプロイ停止や外部APIの401などが重なると、1本のCI/CDにすべてを依存した構成は急に脆くなります。

この記事では、実際に複数の障害が重なったときに見直した「投稿経路の分離」と「検証の段階化」をまとめます。

ポイントは、**ビルド・配信・公開確認・成果確認を別の状態として扱う**ことです。

## 1. まず「成功」を4段階に分ける

自動投稿では、次の4つを同じ「成功」にしない方が安全です。

1. **生成成功**: Markdownや画像が正しく生成された
2. **送信成功**: 外部APIが201/200などを返した
3. **公開成功**: 公開URLを第三者が取得できる
4. **目的達成**: クリック、申込、売上などの成果が管理画面で確認できる

たとえばAtomPubで201 Createdが返っても、それだけで「読者がアクセスできる公開ページまで確認済み」とは限りません。逆に、CIのjobがgreenでも、最後に呼んだrevalidation APIがJSON本文でエラーを返していることがあります。

この区別をログと運用ルールに落とすだけで、かなり誤判定が減りました。

## 2. CIと配信先を疎結合にする

最初の構成は、ざっくり次のような形でした。

```text
Git push
  ↓
GitHub Actions
  ├─ frontmatter検証
  ├─ 画像生成
  ├─ Qiita投稿
  ├─ DEV.to投稿
  ├─ Hatena投稿
  ├─ Zenn同期
  └─ Vercel revalidate
```

便利ですが、Actionsが止まると全経路が止まります。

そこで、配信先を3種類に分けて考えます。

```text
A. CI経由
   GitHub Actions → API投稿

B. 外部API直結
   ローカル/別実行環境 → AtomPubなど

C. Git連携
   GitHub repository → 配信サービス側のGit連携
```

重要なのは、Aが止まってもBかCを残すことです。

特にZennのようにGitHubリポジトリとの連携を持てるサービスでは、記事ファイルを対象リポジトリへ直接pushできれば、重い自前runnerを通さずに公開経路を維持できます。

## 3. 重い処理と「1記事を出す処理」を分離する

無料枠を節約するうえで、最も効きやすいのはworkflowを軽くすることです。

たとえば、1記事を投稿したいだけなのに毎回これを行うと、消費時間が増えます。

- OSパッケージ更新
- ffmpegインストール
- 数百MBの依存関係復元
- 全記事スキャン
- 全画像変換
- 全配信先への重複確認

投稿workflowとメディア生成workflowを分けます。

```yaml
name: Publish article

on:
  push:
    paths:
      - "content/**/*.md"

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: node scripts/publish.mjs
```

動画からGIFを作る処理などは、対象ファイルが変わったときだけ別workflowで走らせます。

```yaml
on:
  push:
    paths:
      - "public/movies/**"
```

これだけでも「記事の文言を1行直しただけでffmpegから再実行」という無駄を避けられます。

## 4. `curl` が0終了でもAPI成功とは限らない

運用中に特に危険だと感じたのがこれです。

```bash
curl -X POST "$URL"
echo "Revalidation triggered successfully"
```

HTTPサーバーが400系のJSONを返していても、`curl`自体は通信できれば0で終了することがあります。そのまま次の`echo`へ進むと、Actions上は成功に見えます。

最低限、HTTPエラーをexit codeへ反映します。

```bash
curl --fail-with-body -sS \
  -X POST \
  "$URL"
```

さらにAPIが`200 OK`で論理エラーを返すタイプなら、JSONも検証します。

```bash
response="$(curl --fail-with-body -sS -X POST "$URL")"

echo "$response" | jq -e '.revalidated == true' >/dev/null
```

ここで大事なのは、**「コマンドが実行された」ではなく「期待した状態になった」を成功条件にする**ことです。

## 5. 外部APIのログは「秘密を出さず、状態は残す」

外部APIの診断ログには次を残すと復旧しやすくなります。

- HTTP status
- request先のサービス名
- 記事slug / 内部ID
- response bodyの安全な範囲
- 公開URL（取得できた場合）

逆に、以下は出しません。

- API key
- access token
- OAuth secret
- WSSE生成元の秘密値

たとえばAtomPubなら、成功時にLocationヘッダーやentry IDを記録しておくと、後から「送信できたか」をGitHub Actionsの過去ログだけで確認できます。

```js
const location = response.headers.get('Location')
console.log(`created: status=${response.status} location=${location ?? '(none)'}`)
```

秘密値をマスクしながら、**公開物を追跡できる識別子だけ残す**のがコツです。

## 6. 401は同じ認証情報で再試行しない

429や5xxはバックオフ再試行が有効ですが、401 Unauthorizedは別です。

```text
429 / 5xx
  → 時間を空けて再試行

401 / 403
  → 権限・token・scope・アプリ設定を見直す
```

401に対して同じ資格情報を何度投げても、成功確率はほぼ上がりません。API利用枠やログだけ消費します。

再試行関数も、ステータスごとに分類します。

```js
if (res.status === 401 || res.status === 403) {
  throw new Error(`auth_error:${res.status}`)
}

if (res.status === 429 || res.status >= 500) {
  // retryable
}
```

## 7. 配信対象をfrontmatterで明示する

複数プラットフォームへ配信するとき、全記事を全部のサービスへ流すのは危険です。

たとえば技術記事と、個人ブログ向けの記事では適した掲載先が違います。

```yaml
---
title: "example"
publish_on: ["hatena"]
draft: false
---
```

配信コード側では、許可された配信先だけ処理します。

```js
function shouldPublish(post, platform) {
  return Array.isArray(post.publish_on)
    && post.publish_on.includes(platform)
}
```

これで「技術コミュニティへ意図しない販促記事を流す」「同じ記事を全媒体へ重複投稿する」といった事故を減らせます。

## 8. 公開確認はAPI送信と別にする

理想は、投稿APIの成功後に次を検証することです。

```text
POST成功
  ↓
公開URL取得
  ↓
GET 200確認
  ↓
タイトル確認
  ↓
必要な表示・リンク確認
```

Node.jsなら概念的には次のようにできます。

```js
const page = await fetch(publicUrl, {
  redirect: 'follow',
  headers: { 'User-Agent': 'publish-verifier/1.0' }
})

if (!page.ok) {
  throw new Error(`public page HTTP ${page.status}`)
}

const html = await page.text()
if (!html.includes(expectedTitle)) {
  throw new Error('expected title not found')
}
```

ここまで通って初めて「公開ページを確認済み」と言えます。

検索エンジンへの反映はさらに後なので、検索結果に出ないことを公開失敗と誤判定しないことも重要です。

## 9. 目的指標は配信ログではなく管理画面で確認する

最後に、公開の成否とビジネス上の成果を分離します。

```text
記事を公開できた
≠
クリックされた
≠
売上が発生した
≠
成果が確定した
```

成果を追うなら、最終的には広告・アフィリエイト・決済サービスなど、**成果を確定する当事者の管理画面や公式レポート**を証拠にします。

CIログだけを見て「収益化成功」と判定しないように、チェックリストを分けておくと安全です。

## まとめ

GitHub Actionsは便利ですが、配信基盤そのものにしてしまうと、無料枠・runner・GitHub側の障害が単一障害点になります。

今回の見直しで特に重要だったのは次の5点でした。

- CI、API送信、公開確認、成果確認を別ステータスにする
- 重いメディア処理を投稿workflowから分離する
- Git連携や外部API直結をフェイルオーバー経路として残す
- 401と429/5xxを同じ再試行ロジックで扱わない
- greenなjobではなく、最終的な外部状態を検証する

自動化を増やすほど、「自動化が止まったときの手動/別経路」も同時に設計しておくと復旧が速くなります。

---

実際の運用ログは個人ブログにも残しています。広告・PRを含む記事があります。

https://beachone1155.hatenablog.com/
