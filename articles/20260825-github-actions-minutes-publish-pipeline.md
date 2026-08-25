---
title: "GitHub Actionsの2,000分を使い切って分かった、記事自動投稿CIの「無駄な実行」を削る方法"
emoji: "⏱️"
type: "tech"
topics: ["githubactions", "cicd", "automation", "zenn"]
published: true
---

## はじめに

GitHub Actionsで、1つのMarkdownから複数の投稿先へ記事を配信する仕組みを運用しています。

便利な一方で、private repository の GitHub Actions 利用枠を使い切り、新しいジョブに runner が割り当たらなくなりました。そこでワークフローを見直すと、**「今回の変更には不要な処理」まで毎回起動していた**ことが分かりました。

この記事では、実際に直したポイントを、汎用化してまとめます。

> GitHub Free の private repository では GitHub-hosted runner の月間利用枠があります。料金・利用枠は変更される可能性があるため、最新値は公式ドキュメントを確認してください。

## 1. 重い前処理を「毎回」実行しない

以前は記事の変更内容に関係なく、毎回FFmpegのインストールと動画→GIF生成を実行していました。

```yaml
- name: Install FFmpeg
  run: sudo apt-get update && sudo apt-get install -y ffmpeg

- name: Generate GIFs from movies
  run: pnpm run generate-gifs
```

動画を含まない記事でもこの処理が走るため、積み重なると無視できません。

今回の修正では、通常の記事公開パスから常時実行を外しました。必要なら、変更ファイルに動画が含まれる場合だけ起動する条件付きジョブに分離するのが次の改善です。

## 2. 全記事を走査せず「変更された記事だけ」投稿する

マルチ投稿では、投稿先ごとに毎回全記事を走査するより、pushで変わったMarkdownだけに絞る方が安全です。

```bash
BEFORE_SHA="${{ github.event.before }}"

mapfile -t changed_files < <(
  git diff --name-only "$BEFORE_SHA" HEAD -- content \
    | grep -E '^content/.*\.md$' || true
)

if [ "${#changed_files[@]}" -eq 0 ]; then
  echo "No changed Markdown articles."
  exit 0
fi

for file in "${changed_files[@]}"; do
  slug="$(basename "$file" .md)"
  node scripts/publish.mjs "$slug"
done
```

ポイントは、`grep` が0件でもワークフロー全体を失敗させないことと、投稿スクリプトへ明示的にslugを渡すことです。

手動実行時は差分の基準SHAがない場合があるので、`publish_slug` のような入力を必須にして、意図せず全記事を再投稿しないようにしました。

## 3. 投稿先ごとに必要なsecretだけ渡す

Qiitaの投稿ステップにDEV.toやXのsecretまで渡す必要はありません。

```yaml
- name: Publish changed articles to Qiita
  env:
    QIITA_TOKEN: ${{ secrets.QIITA_TOKEN }}
    PUBLISH_TARGETS: qiita
```

投稿先単位でsecretを絞ると、設定が読みやすくなるだけでなく、将来ワークフローを分割するときにも扱いやすくなります。

## 4. 既知の失敗をpushのたびに再試行しない

外部APIの認証エラーなど、原因が分かっていて資格情報の修正待ちになっているジョブを、通常pushのたびに自動実行すると利用時間だけ消費します。

今回は、HTTP 401が続いていたX投稿ワークフローの自動pushトリガーを止め、資格情報を直すまでは `workflow_dispatch` の手動実行だけにしました。

```yaml
on:
  workflow_dispatch:
```

「直るまで自動再試行しない」は地味ですが、外部サービス連携が多いCIでは効果があります。

## 5. HTTP 200だけで「成功」にしない

再検証用APIなどでは、HTTP自体は成功していても本文がエラーJSONというケースがあります。

以前は `curl` が0で終了すれば成功扱いでしたが、レスポンス本文も確認するようにしました。

```bash
response="$(curl --fail-with-body --silent --show-error \
  -X POST "${VERCEL_URL}/api/revalidate?secret=${REVALIDATE_SECRET}&path=/blog")"

echo "$response"

if printf '%s' "$response" | grep -q '"error"'; then
  echo "Revalidation returned an error payload." >&2
  exit 1
fi
```

CIの緑色だけを見るのではなく、**「そのチェックが本当に成功条件を検証しているか」**まで見る必要があります。

## 6. public repository を「節約目的だけ」で使わない

GitHubの標準GitHub-hosted runnerは、public repositoryでは無料で利用できる範囲があります。一方で、private repositoryではプランに応じた利用枠を消費します。

ただし、Actions時間を節約するためだけにprivateなコードやsecretをpublicへ移すのは本末転倒です。

公開して問題ないコンテンツ（たとえばZennの記事リポジトリ）の同期はpublic repoで行い、秘密情報が必要な公開処理はprivate repoに残す、というように責務で分ける方が安全です。

## 7. 次にやるなら「対象判定ジョブ」を先頭に置く

今回の修正は、不要な全記事走査や重い前処理を減らすところまでです。

さらに詰めるなら、最初に軽量なジョブで変更内容を判定して、必要な投稿先だけmatrixへ出す構成にします。

```text
changed files
   ↓
detect targets
   ├─ qiita only
   ├─ hatena only
   ├─ dev.to only
   └─ no publish → stop
```

この形なら、Hatenaだけの記事変更でQiita用の画像準備ジョブを起動する、といった無駄も避けやすくなります。

## まとめ

今回いちばん大きかったのは、「1回あたり数十秒〜数分だから気にしない」が積み重なることでした。

- 重い前処理を常時実行しない
- 変更された記事だけを対象にする
- 投稿先ごとにsecretを絞る
- 既知の失敗は自動再試行しない
- HTTP成功と業務上の成功を分けて検証する
- public/privateはコストではなく公開可否で分ける

記事自動投稿のようにpush回数が多い仕組みほど、**「何を実行するか」より先に「今回は何を実行しなくてよいか」を判定する**のが効きます。

## 参考

- GitHub Docs: About billing for GitHub Actions
  https://docs.github.com/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- GitHub Docs: GitHub-hosted runners
  https://docs.github.com/actions/using-github-hosted-runners/about-github-hosted-runners
