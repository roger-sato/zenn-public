---
title: "Zennの記事の画像をいい感じに管理するようにした"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Zenn", "ZennCLI"]
published: true
---

[この記事](https://zenn.dev/lovablepleiad/articles/zenn_good-slug-23y01m12d)を少し改良した話になります。

Zenn ではリポジトリに画像を入れて、記事内から参照できます。(参考:[https://zenn.dev/zenn/articles/deploy-github-images](公式記事))
ルートディレクトリに images ディレクトリを作って、その中に画像ファイルを入れるだけでいいのですが
記事の数が多くなると管理が煩雑になります。

そこで、楽にいい感じに管理できるようにしたいなぁというのが動機です。
ざっくりとした要件は次のとおりです。

- images ディレクトリ配下にサブディレクトリなどを作って管理しやすくする
- 記事作成時にサブディレクトリなどが勝手にできる
- 使ってないサブディレクトリはまとめて削除できるようにする

最終的には`package.json`のスクリプトを次のようにしました。

```json
{
  "name": "zenn",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "preview": "npx zenn preview",
    "new:article": "slug_pre=$(git rev-parse --abbrev-ref head) && npx zenn new:article --slug ${slug_pre}-$(date +%yy%mm%dd) && mkdir -p images/$slug_pre",
    "clean:images": "find ./images -depth -type d -empty -exec rmdir {} \\;"
  },
  "keywords": [],
  "author": "",
  "license": "isc",
  "dependencies": {
    "zenn-cli": "^0.1.134"
  }
}
```

`new:article`は新しく記事を作るときに実行されるもので、記事と画像のサブディレクトリが作られます。
記事を作る前に git branch を切っておく必要があります。

```sh
$ git checkout -b example-article

$ npm run new:article
created: articles/example-article-23y05m12d.md

$ tree
.
├── README.md
├── articles
│   └── example-article-23y05m12d.md
├── books
├── images
│   └── example-article
├── package-lock.json
└── package.json

```

あとは、作られた`images/example-article`の下に画像を入れて、記事からこんな感じに参照できます。

```markdown
![](/images/example-article/sample.png)
```

ただ、画像を使わない記事でもディレクトリを作ってしまうので、images 配下に空のディレクトリが溜まってしまいます。
そこで、空のディレクりを消すスクリプトも追加しました。

```sh
$ npm run clean:images
```

images ディレクトリ配下で空のディレクトリを消してくれます。
