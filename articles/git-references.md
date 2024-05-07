---
title: "Gitの参照周りについて整理する"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Git"]
published: true
---

Gitの参照周りの概念や用語があまり理解できていなかったので整理してみようと思った。

## まとめ

![Git References](/images/git-references/git-references.png)

## Commit Hash

- コミットの識別子
- 一番上のTreeオブジェクトのSHA1ハッシュ地親コミット、Author、Commiter、タイムスタンプを含んだデータのSHA1バッシュ地
  - タイムスタンプを含むので同じデータに対するコミットでも時刻によりハッシュ値は違うものになる
- Git公式の用語はなさそう[^1]
  - GitHubではCommit ID[^2]と呼ばれる
- 例

```bash
dc948eb27d5150c72b09750e856c015024666bd1
```

- `git rev-parse` コマンドでrefをcommit hashとして解決してくれる

```bash
$ git rev-parse main
dc948eb27d5150c72b09750e856c015024666bd1
```

### 参考

- [Git - Git Objects (git-scm.com)](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)

[^1]: [gitで特定のcommitバージョン/リビジョンを指すコレをなんと呼ぶか問題](https://qiita.com/bigwheel/items/0b331451558637ee29b3)
[^2]: [GitHub 用語集](https://docs.github.com/ja/get-started/learning-about-github/github-glossary#%E3%82%B3%E3%83%9F%E3%83%83%E3%83%88-id)

## Reference

- Commit Hashへのエイリアス
- `ref`と呼ばれる
- 実態は `.git/refs` に保存されている

```bash
.git/refs/
  -- heads/          ローカルブランチ refs/heads/mainなど
  -- remotes/origin/ リモートブランチ refs/remotes/origin/mainなど
  -- tags/           タグ refs/tags/v0.0.1など
```

### Short name

単にmainやHEADと指定するとGitは `${GIT_DIR}/<refname>` > `refs/<refname>` > `refs/tags/refname` > `refs/heads/<refname>`> `refs/remotes/<refname>` という順番で同名のファイルがないかを探して見つかるとそのrefとして解決してくれる。

- `git rev-parse --symbolic-full-name`でどういう参照として解決されるかが確認できる

```bash
# ローカルブランチ
$ git rev-parse --symbolic-full-name main
refs/heads/main

# リモートブランチ
$ git rev-parse --symbolic-full-name  origin/main
refs/remotes/origin/main

# タグ
$ git tag
v0.0.1
$ git rev-parse --symbolic-full-name v0.0.1
refs/tags/v0.0.1
```

## Packed ref

- `.git/refs`ディレクトリの内容を圧縮したもの
- レポジトリが大きいと `git gc` の際に圧縮されることがある
- 能動的に実行するには `git pack-refs --all` を実行する（unpackする方法はgitでは提供されていない）

## Special ref

- 主にGitが内部で利用している特殊な参照
- `.git`以下にファイルとして保存されるが、内容はそれぞれの種類ごとに異なる
- 普段の作業でユーザが利用するのは`HEAD`くらい
- 例
  | 参照 | 概要 |
  | --- | --- |
  | HEAD | 現在のブランチ。これが指しているコミットに新たなコミットが追加される。通常はSymbolic refだが、detached状態の場合はCommit Hashになる。 |
  | FETCH_HEAD | 直近でフェッチされたブランチ。`git pull` = `git fetch` + `git merge FETCH_HEAD`|
  | ORIG_HEAD | git merge や git rebase など大きな変更がある場合に直前のHEADの位置を保存するために作られる。`git reset --hard ORIG_HEAD`で元に戻すことができる。 |

## Symbolic ref

- 他のrefを指すref
- HEADは通常symbolic ref
- 例

```bash
$ cat .git/HEAD
ref: refs/heads/main
```

## Refspec

- リモートサーバー上のブランチとリモート追跡ブランチ（remote-tracking branch）をマッピングを表記するフォーマット

```bash
[+]<src>:<dst>
```

という形式をしている。`<src>` はリモートサーバー上のブランチ、 `<dst>` はリモート追跡ブランチを指す。 fetchしたときにデフォルトではfast-forwardマージができないとfetchが失敗するが、 `+` をつけると強制的にリモート追跡ブランチを更新する。

:::details Gitのブランチのローカルとリモート
あるブランチには3種類ある
(1) ローカルブランチ ローカルマシン上の `refs/heads/`
(2) リモート追跡ブランチ ローカルマシン上の `refs/remotes/<origin name>/`
(3) リモートサーバ上のブランチ サーバマシン上の `refs/heads/`
Refspecは(2)と(3)のマッピングを定義するもの。(1)と(2)のマッピングはconfigの `branch.<branch-name>.merge` で定義される。
:::

`git clone` や `git remote add origin <url>`した際に

```bash
fetch = +refs/heads/*:refs/remotes/origin/*
```

という設定が `.git/config` に書き込まれるが、これはfetchしたときにすべてのリモートサーバ上のブランチをリモート追跡ブランチとしてfetchするという意味になる。

### 参考

- [git - What are the differences between local branch, local tracking branch, remote branch and remote tracking branch? - Stack Overflow](https://stackoverflow.com/questions/16408300/what-are-the-differences-between-local-branch-local-tracking-branch-remote-bra)
- [Gitの内側 - Refspec](https://git-scm.com/book/ja/v2/Git%E3%81%AE%E5%86%85%E5%81%B4-Refspec)
- [Git - リモートブランチ](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%83%96%E3%83%A9%E3%83%B3%E3%83%81%E6%A9%9F%E8%83%BD-%E3%83%AA%E3%83%A2%E3%83%BC%E3%83%88%E3%83%96%E3%83%A9%E3%83%B3%E3%83%81)

## Relative ref

- あるコミットからの相対的なコミットを参照するref
- `~{n}` では第一の親コミットをn番目前までたどる
- mergeによって親コミットが複数あるときに、第一の親コミット以外を選択したい場合は `^{n}` で第nの親を選択する
- 例

```text
HEAD~1 # HEADが指すコミットから1つ前のコミット（2つ親がある場合は第一の親コミット）
HEAD^2 # HEADが指すコミットの第二の親コミットを指す
main^2~2 # HEADが指すコミットから1つ前のさらに2つ前のコミット（第二の親ルート）

Gに対して
 - Dは第一の親コミット
 - Fは第二の親コミット

 HEAD^2~2  HEAD~1
 HEAD~3    HEAD^1
     ▼     ▼
--A--B--C--D--G <--HEAD, main
      \      /
       E----F <-- HEAD^2
```

### 参考

- [What's the difference between HEAD^ and HEAD~ in Git? - Stack Overflow](https://stackoverflow.com/questions/2221658/whats-the-difference-between-head-and-head-in-git)
- [git-rev-parse - Pick out and massage parameters](https://git-scm.com/docs/git-rev-parse#Documentation/git-rev-parse.txt-emltrevgtemegemHEADv1510em:~:text=Parent%20commits%20are%20ordered%20left%2Dto%2Dright)

## Reflog

- HEADの変更履歴
- コミットやブランチを切り替えた際などに更新される
- `git reflog` コマンドで履歴を確認できる
- `HEAD@{n}` の形式でnはHEADの変更履歴でn番目前の変更を表す
- 例

```bash
$ git reflog
24aab06 (HEAD -> main) HEAD@{0}: merge topic: Merge made by the 'ort' strategy.
61ecf98 HEAD@{1}: commit: C
0146958 HEAD@{2}: checkout: moving from topic to main
7de4317 (topic) HEAD@{3}: commit: F
```

- `git reset` で誤ったそうさを戻すのに利用できる
  - [【git】git reflogでgitの履歴に対する過去の操作を管理する - Ren's blog (hatenablog.com)](https://rennnosukesann.hatenablog.com/entry/2018/03/06/231439)

## Revision

- これ自体はRefではなく、Commit HashやRefを使ったクエリシステムのようなもので、Gitオブジェクトを参照できる記法
- 単一のコミットだけでなく複数のコミットを指したりもする（Refは単一のコミットに対するエイリアス）
- `git rev-parse` コマンドでどのようなCommit Hashとして解決されるかが確認できる

### 例

- **Short SHA-1**
  - 他に同じプレフィックスをもつSHA1が存在しなければ、SHA1ハッシュの一部でもフルSHA1ハッシュとして解決される

```bash
$ git rev-parse ae13719
ae137196659150446b94f8dacc5549d50ccf8414
```

- Commit Ranges
  - `git log`や`git rev-list`はリビジョンを受け取るとそこからたどり着けるコミットの一覧を返す
  - `<rev>`
    - revからたどり着けるコミットのリスト
  - `^<rev>`
    - revからたどり着けるコミットを除外する
  - Double dot `<rev1>..<rev2>`
    - rev1からはたどり着けず、rev2からのみたどり着けるコミットのリスト
    - `^<rev1> <rev2>` と同じ
  - Triple dot `<rev1>...<rev2>`
    - rev1もしくはrev2のどちらかしかからたどり着けないコミットのリスト

### 参考

- [How do Git revisions and references relate to each other? - Stack Overflow](https://stackoverflow.com/questions/73145810/how-do-git-revisions-and-references-relate-to-each-other)

## おわりに

Gitの多くのコマンドでコミットを参照することがあるが、コミットのSHA1ハッシュ値以外にも参照の仕方をしてっいると役立つと思う。

## 全体の参考

- [Git Refs: What You Need to Know | Atlassian Git Tutorial](https://www.atlassian.com/git/tutorials/refs-and-the-reflog)
- [Git - revisions Documentation (git-scm.com)](https://git-scm.com/docs/revisions)
- [Git - Revision Selection (git-scm.com)](https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection)
