---
title:      "サクッとPullRequestを開くための便利なコマンドの紹介"
date:       2020-02-02T21:00:00+09:00
categories: ["git"]
tags:       ["tech"]
---

コードを読んでいると関連したPull Requestを見に行きたいということはよくあると思います.

特にCREとしてプロダクトのコードを読む時には, どのような背景でこのような実装をしているのかを調べたいことが多く, PRの議論を参考にすることが多いです.

今回はコードを読んでPRを開くという操作が少しでも楽になるような設定を紹介しようと思います.

## コードからPull Requestを見つける

[Commit Hash から、該当 Pull Request を見つける方法](https://qiita.com/awakia/items/f14dc6310e469964a8f7)の設定を拝借して利用しています.

```
# .gitconfig
[alias]
  showpr = "!f() { git log --merges --oneline --reverse --ancestry-path $1...master | grep 'Merge pull request #' | head -n 1; };f"
  openpr = "!f() { hub browse -- `git log --merges --oneline --reverse --ancestry-path $1...master | grep 'Merge pull request #' | head -n 1 | cut -f5 -d' ' | sed -e 's%#%pull/%'`; };f"
```

この二つの設定を追加することで以下のようにPull Requestを検索することができます.

```
$ git showpr 753f8c4a
0000000000 Merge pull request #1 from XXX/XXX # こんな感じでPR番号が表示される
$ git openpr 753f8c4a
-> ブラウザで該当のPRを開く
```

僕は普段Vimでコードを読み書きするため, [tpope/vim-fugitive](https://github.com/tpope/vim-fugitive)を使ってCommit Hashを検索しています.

コードを読みながらこの行・このファイルのPRを探したいなという時に:GblameでCommit Hashを取得します.
あとはCommit Hashをコピペしてgit openpr XXXXとするだけでPRを開くことができます.

{{< figure src="/img/hub_useful_command/1.png" class="center" >}}

### 開きたいファイルが決まっている場合

また, 開きたいファイルが決まっている時はVimを起動せずにgit blameの結果からPRを開くということもできるようにしています.

そのためのaliasが以下のようになります.
最近fzfを利用しており, git blameの結果をfzfで選択し, 選択した行のCommit Hashを先ほどのopenprコマンドに流しています.

```
# .gitconfig
[alias]
  blamepr = "!f() { git blame $1 | fzf | cut -f 1 -d ' ' | xargs -I@ git openpr @; };f"
```

```
$ git blamepr filename
```

このコマンドを実行すると'git blame'の結果が表示されるので, PRを開きたい行を選択するとブラウザに遷移します.

{{< figure src="/img/hub_useful_command/2.png" class="center" >}}

このような細かい改善は毎日積もり積もって大きな効果を発揮すると思います.
まだまだ改善できる箇所は多いので今後もさらに便利にしていきたいなと思います.

今回この記事を書く中で定期的にくるカスタマイズしたい欲が湧いてきたのでまた何か良い設定を見つけた時には紹介できればと思います.
