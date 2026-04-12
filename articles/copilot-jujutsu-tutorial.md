---
title: "GitHub CopilotにJujutsu（jj）のチュートリアルをやらせてみた"
emoji: "🤖"
type: "tech"
topics:
  - "jujutsu"
  - "jj"
  - "git"
  - "githubcopilot"
  - "vcs"
published: true
---

:::message
この記事は GitHub Copilot CLI（Claude Sonnet 4.6 搭載）に依頼して書いてもらった下書きをベースにしています。チュートリアルの実施・記録・この記事の執筆まで、すべて AI が行いました。感想部分も AI による所感です。
:::

# GitHub Copilot に Jujutsu のチュートリアルをやらせてみた

次世代 VCS として注目されている **Jujutsu（jj）** が気になっていたので、GitHub Copilot CLI に公式チュートリアルをそのままやらせてみました。コマンドの実行・記録・この記事の執筆まで、すべて AI に任せています。

- **参照チュートリアル**: https://jj-vcs.github.io/jj/latest/tutorial/
- **実行環境**: jj 0.37.0（Linux）
- **使用 AI**: GitHub Copilot CLI（Claude Sonnet 4.6）

---

## Jujutsu（jj）とは

Jujutsu は Google 社員が開発した次世代 VCS です。Git リポジトリとの互換性を保ちつつ、Git のつらい部分を設計レベルで解消しようというコンセプトで作られています。

---

## やったこと

### 1. クローン

Git リポジトリをそのままクローンできます。

```sh
$ jj git clone https://github.com/octocat/Hello-World
Fetching into new repo in "/tmp/Hello-World"
bookmark: master@origin          [new] tracked
bookmark: octocat-patch-1@origin [new] untracked
bookmark: test@origin            [new] untracked
Setting the revset alias `trunk()` to `master@origin`
Working copy  (@) now at: nxpmromt 398df47c (empty) (no description set)
Parent commit (@-)      : orrkosyo 7fd1a60b master | (empty) Merge pull request #6 from Spaceghost/patch-1
```

注目ポイントは最後の 2 行です。クローン直後から「ワーキングコピー」がコミットとして作られています。これが jj の根本的な違いです。

### 2. 状態確認と Change ID / Commit ID

```sh
$ jj st
The working copy has no changes.
Working copy  (@) : nxpmromt 398df47c (empty) (no description set)
Parent commit (@-): orrkosyo 7fd1a60b master | (empty) Merge pull request #6 from Spaceghost/patch-1
```

jj には識別子が **2 種類**あります。

| 識別子 | 例 | 特徴 |
|--------|-----|------|
| **Change ID** | `nxpmromt` | コミット内容を書き換えても変わらない安定した ID |
| **Commit ID** | `398df47c` | Git のハッシュ相当。内容が変わるたびに更新される |

Git ユーザーが慣れるまでに少し時間がかかる概念ですが、「変更そのもの」と「その時点のスナップショット」を別々に追跡できるのが特徴です。

### 3. 初めてのチェンジ：`git add` が不要

```sh
$ jj describe -m "Say goodbye"
$ sed -i 's/Hello/Goodbye/' README
$ jj st
Working copy changes:
M README
Working copy  (@) : nxpmromt 6c452c83 Say goodbye
```

**`git add` が不要です。** ファイルを編集するだけで自動的に現在のチェンジに取り込まれます。

チェンジが完成したら `jj new` で次のチェンジへ進みます（Git でいう `git commit` に相当）。

```sh
$ jj new
Working copy  (@) now at: zqroszks f4ad6a8f (empty) (no description set)
Parent commit (@-)      : nxpmromt 6c452c83 Say goodbye
```

### 4. revset：柔軟な履歴参照

`jj log` では revset と呼ばれる独自の式言語でコミットを絞り込めます。

```sh
$ jj log -r '@ | root() | bookmarks()'
```

`|`（和集合）、`&`（積集合）、`~`（差集合）などの演算子を組み合わせて、見たいコミットだけを表示できます。強力ですが、最初は覚えることが多いです。

### 5. コンフリクットがコミットに記録される

これが jj の最大の特徴のひとつです。A → B1 → B2 → C というコミットチェーンを作り、B2 を B1 を飛ばして A の直上にリベースします。

```sh
$ jj rebase -s B2の変更ID -o AのチェンジID
Rebased 2 commits to destination
Working copy  (@) now at: tpytuwrw 85c87089 (conflict) C
Warning: There are unresolved conflicts at these paths:
file1    2-sided conflict
New conflicts appeared in 2 commits:
  tpytuwrw 85c87089 (conflict) C
  uuqkworq 2c32480e (conflict) B2
```

**rebase がコンフリクットで止まりません。** コンフリクットはコミットに記録され、作業は続行されます。子孫の C まで一緒にリベースされているのもポイントです。

解消するには B2 の上に新しいチェンジを作り、ファイルを修正して `jj squash` するだけです。

```sh
$ jj new B2のチェンジID
$ echo resolved > file1
$ jj squash
Existing conflicts were resolved or abandoned from 2 commits.
```

別ファイルしか触っていない C のコンフリクットも**自動的に解消**されました。

### 6. `jj op log` と `jj undo`

すべての操作が記録されています。

```sh
$ jj op log
@  b363511a18fc  squash commits into ...  args: jj squash
○  22e47476a2e9  snapshot working copy   args: jj status
○  2bc02892b402  new empty commit        args: jj new ...
...
```

```sh
$ jj undo
Restored to operation: 22e47476a2e9 (snapshot working copy)
```

squash が即座に取り消され、コンフリクットが復活しました。Git の reflog より直感的で、どんな操作でも 1 コマンドで戻せます。

### 7. `jj edit` で過去のコミットを直接修正

abc → ABC → ABCD というチェーンを作り、ABC で `c` の大文字化をうっかり忘れた例です。

```sh
$ jj edit ABC のチェンジID
$ printf 'A\nB\nC\n' > file
```

これだけで ABC の内容が更新され、子孫の ABCD は自動的にリベースされて diff が正しく整理されます。`git rebase -i` を使わずに済みます。

---

## AI（Copilot）の所感

チュートリアルを実施して感じた Git との違いをまとめます。

### 良かった点

**1. `git add` がない**
ファイルを編集するだけで反映されるのは、地味ですが体験として大きく違います。「ステージし忘れてコミットした」という事故がなくなります。

**2. コンフリクットへの恐怖感が減る**
「rebase 中にコンフリクットが起きると作業が中断される」という Git の挙動は、複雑な履歴操作をためらわせる原因のひとつです。jj ではコンフリクットがコミットに記録されたまま処理が続行するため、心理的ハードルが下がります。

**3. `jj op log` + `jj undo` が強力**
あらゆる操作が記録されており、1 コマンドで直前の状態に戻せます。「壊してしまった」という不安なく履歴操作を試せます。

**4. Change ID で「あの変更」を追いやすい**
`git commit --amend` のたびにハッシュが変わる Git と違い、Change ID は安定しているため「さっき作った変更」を常に同じ ID で参照できます。

### 難しかった点

- **新概念が多い**: Change ID / revset / bookmark など、最初に覚えることが Git より多い
- **エコシステムがまだ発展途上**: GitHub との連携は Git 経由が前提で、ネイティブ jj リポジトリのクローンは未対応
- **ツールの対応**: エディタの統合やCI等の対応状況はGitに比べてまだ限定的

---

## まとめ

| 概念 | Git | jj |
|------|-----|----|
| ステージング | `git add` が必要 | 自動 |
| コミット識別子 | ハッシュのみ | Change ID（安定）+ Commit ID |
| コンフリクット | rebase を止める | コミットに記録して続行 |
| 過去コミットの編集 | `git rebase -i` | `jj edit` |
| 操作の取り消し | reflog（手間あり） | `jj undo`（1コマンド） |
| ブランチ | branch | bookmark |

jj は「Git の操作ミスによる事故を設計レベルで減らす」という方向性がはっきりしており、特に**コミット履歴を丁寧に整理したい開発者**には魅力的な選択肢になりそうです。

Git との互換性を保ったまま使えるので、既存リポジトリに試してみるハードルも低いです。興味があればぜひ公式チュートリアルを触ってみてください。

---

## 参考リンク

- [Jujutsu 公式チュートリアル](https://jj-vcs.github.io/jj/latest/tutorial/)
- [Steve Klabnik によるチュートリアル（より網羅的）](https://steveklabnik.github.io/jujutsu-tutorial/)
- [revset ドキュメント](https://jj-vcs.github.io/jj/latest/revsets/)
- [GitHub Copilot CLI](https://docs.github.com/copilot/concepts/agents/about-copilot-cli)
