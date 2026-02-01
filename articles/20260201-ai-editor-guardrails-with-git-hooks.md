---
title: "GitHub Organizationに課金できなくても、AIエディタの暴走は止められる（pre-commit / pre-push + .cursor/rules）"
emoji: "📝"
type: "tech"
topics: ["ai","git","cursor","workflow"]
published: true
---


# GitHub Organizationに課金できなくても、AIエディタの暴走は止められる（pre-commit / pre-push + .cursor/rules）

## はじめに（ある日、AIが気持ちよく“仕事”しすぎた）

CursorとかWindsurfとか、最近のAIエディタって、放っておくと本当に手が速いですよね。
気づいたらファイルが増え、依存が増え、コマンド履歴が増え、そして「そのまま push しそう」になる。

でも、GitHubのOrganizationで **Branch protection をちゃんと課金して運用**するのが難しい状況だと、
最後の砦が薄くて、心理的にけっこう怖い。

なので私は、ローカル側で“物理ロック”をかけました。
やりたいのはこれだけです：

- **main / develop ではコミットさせない**（そもそもAIが間違えた場所で作業しない）
- **main / develop へは push させない**（最悪でもローカルで止まる）
- **AIには守るべきプロジェクトルールを常に読ませる**（毎回言い聞かせない）

この記事は、そのための最小構成の話です。

## 結論：ガードレールは「二段」にすると強い

私はこの2つを重ねるのが一番ラクでした。

- **`.cursor/rules`**: “AIの意思決定”を縛る（暴走しにくい道を作る）
- **Gitフック（pre-commit / pre-push）**: “最後の操作”を物理的に止める（暴走しても落ちる）

`.cursor/rules` は「お願い」寄り、Gitフックは「強制」寄り、というイメージです。

## 1) pre-commit：main/develop での直接コミットを禁止する

まずは `pre-commit`。
コミットの直前に実行されるフックで、ブランチ名を見て止めます。

私が使っているのはこんな感じ（ほぼそのまま貼ります）：

```bash
#!/bin/bash

# 現在のブランチ名を取得
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\\(.*\\),\\1,')

# main または develop ブランチならコミット・マージを阻止
if [ "$current_branch" = "main" ] || [ "$current_branch" = "develop" ]; then
    echo "----------------------------------------------------"
    echo "🔴 ERROR: ローカルの '$current_branch' ブランチでの直接コミット/マージは禁止されています。"
    echo "   理由: 誤ったブランチへの作業防止のため、フックで制限しています。"
    echo "   対処: 作業用ブランチを切ってから変更してください。"
    echo "   (例: git checkout -b feature/xxx)"
    echo "----------------------------------------------------"
    exit 1
fi

exit 0
```

### どこに置く？

一番雑にやるなら、対象リポジトリの `.git/hooks/pre-commit` に置きます（実行権限も付ける）。

```bash
cp /path/to/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

> [!NOTE]
> `.git/hooks` は Git 管理外です。チームで共有したい場合は `core.hooksPath` を使って、リポジトリ内のディレクトリにフックを置く方式にすると運用が安定します（後述）。

### これで何が嬉しい？

AIが「じゃあコミットしとくね」って勝手にやりだしても、**main/develop なら物理的に落ちる**。
「うっかり」系の事故って、だいたい最後の 1 コマンドで起きるので、ここで止まるのは本当に効きます。

## 2) pre-push：main/develop への直接 push を禁止する

次は `pre-push`。
push 直前に、Gitが stdin で渡してくる情報（どの ref をどこへ push するか）を見て止めます。

```bash
#!/bin/bash

# 保護したいリモートブランチのリスト
PROTECTED_BRANCHES=("main" "develop")

# Gitは pre-push 実行時に、標準入力(stdin)で以下の情報を渡してきます
# <local_ref> <local_sha1> <remote_ref> <remote_sha1>
# これをループで読み取ってチェックします
while read local_ref local_sha remote_ref remote_sha
do
    # remote_ref (例: refs/heads/main) からブランチ名部分だけ抽出
    target_branch=${remote_ref#refs/heads/}

    for protected in "${PROTECTED_BRANCHES[@]}"; do
        if [ "$target_branch" = "$protected" ]; then
            echo "----------------------------------------------------"
            echo "🔴 ERROR: リモートの '$target_branch' への直接プッシュは禁止されています。"
            echo "   理由: GitHub Free版環境のため、フックで保護しています。"
            echo "   対処: Pull Requestを作成してマージしてください。"
            echo "----------------------------------------------------"
            # 1つでも禁止ブランチへのプッシュが含まれていれば停止
            exit 1
        fi
    done
done

exit 0
```

これも `.git/hooks/pre-push` に置けばOKです。

```bash
cp /path/to/pre-push .git/hooks/pre-push
chmod +x .git/hooks/pre-push
```

### これ、AI対策として強いポイント

AIが暴走してる時って、だいたい「差分がデカい」ことよりも「判断が早い」んですよね。
pre-push は **“pushという行為”を止める**ので、差分の内容に関係なく安全装置として働きます。

## 3) `.cursor/rules`：AIの“行動原理”を縛る（お願いではなく習慣にする）

Gitフックは最後の砦ですが、できればそもそも暴走してほしくない。
そこで効くのが `.cursor/rules` です。

このリポジトリだと、例えばこういう方針がすでにルール化されています：

- **`main` / `develop` worktree では AI はコード変更しない**
- **1機能=1 feature ブランチ=1 worktree**
- **push / PR 作成などのリモート影響操作は、ユーザーの明示指示が必要**

これ、地味にめちゃくちゃ効きます。
理由は単純で、AIが「安全に見える次の一手」を選び続けた結果、事故が減るから。

### 追加で入れておくと事故が減る“言い回し”

私は `.cursor/rules` に、次みたいな「禁止」というより「手順」を書くのが好きです。
（AIは“空気”より“手順”に従いやすい）

```md
## ガードレール
- `git checkout main` / `git checkout develop` を提案しない（読むだけにする）
- `git push` / `gh pr create` は、必ず「実行してよいか」を先に確認する
- 変更は feature worktree 内で完結させ、最後に `git status` と差分を提示する
```

> [!TIP]
> 「何をするな」だけだと抜け道を探しがちなので、「代わりに何をするか」まで書くと安定します。

## 4) （運用編）Gitフックを“リポジトリ管理”したい場合の現実的な落としどころ

`.git/hooks` は共有できないので、チームで揃えるなら `core.hooksPath` がラクです。

例として `scripts/githooks/` にフックを置くなら：

```bash
mkdir -p scripts/githooks
cp /path/to/pre-commit scripts/githooks/pre-commit
cp /path/to/pre-push scripts/githooks/pre-push
chmod +x scripts/githooks/pre-commit scripts/githooks/pre-push

git config core.hooksPath scripts/githooks
```

これで、そのリポジトリでは Git が `scripts/githooks` をフックとして参照します。

> [!NOTE]
> `git config core.hooksPath ...` は各ローカルに設定が必要です。READMEに「最初にやること」として書いておくと定着します。

## 5) それでも起きる“AI暴走あるある”と対策メモ

- **勝手にブランチを切り替える**
  - `.cursor/rules` 側で「develop/main には移動しない」を明文化
  - pre-commit が最後に止める
- **勢いで `git push origin HEAD` しようとする**
  - `.cursor/rules` 側で「push は確認必須」
  - pre-push が最後に止める
- **大規模変更を一気に入れる**
  - これはフックよりも運用。`.cursor/rules` に「小さく分ける」「差分を見せる」を書くのが効きます

## まとめ：無料でも“事故は減らせる”。しかも精神が軽い

GitHub側でガチガチに守れない状況でも、ローカルでできることは意外と多いです。
特にAIエディタ相手だと、次の構えが効きました。

- **先に `.cursor/rules` で“安全な道”を作る**
- **最後に Gitフックで“物理ロック”をかける**

「AIが賢いから大丈夫」じゃなくて、「賢いからこそ、速さにブレーキを付ける」。
これで私は、かなり安心して Cursor / Antigravity / Windsurf と付き合えるようになりました。

