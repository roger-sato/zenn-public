---
title: "zennのslugをいい感じにつける"
emoji: "👌"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Zenn"]
published: true
---

slug とは、記事や本に紐付けるユニークな ID です。URL にも利用されるので、基本的には公開後は変更することができません。
Zenn CLI で特に指定しないで記事を作成すると、slug はランダムな値で作成されます。
これはファイルの名前にもなったりするので、ランダムだと後々にメンテナンスが面倒な事になります。

# slug を指定する

[公式記事](https://zenn.dev/zenn/articles/what-is-slug)にも書いてありますが、記事を作成するときに slug オプションで指定することができます。

```shell
npx zenn new:article --slug good-something-slug
# => 'articles/good-something-slug.md'が作成される
```

私は npm の scripts を使うのが好きなので、slug を引数にとって次のようにできそうです。

```json:package.json
{
  ...
  "scripts": {
    "new:article": "npx zenn new:article --slug $npm_config_slug"
  }
  ...
}

```

以下は、先ほどと同じ結果になります。

```shell
npm run new:article -slug=good-something-slug
# => 'articles/good-something-slug.md'が作成される
```

ただ、記事を作成するたびに slug を指定するのは面倒だと感じたので、最終的に git の branch と作成日を組み合わせて指定することにしました。

```json:package.json
{
  "scripts": {
    "new:article": "npx zenn new:article --slug $(git rev-parse --abbrev-ref HEAD)-$(date +%yy%mm%dd)"
  }
}
```

```shell
git checkout -b "good-something-slug"
npm run new:article
# => 'articles/good-something-slug-23y01m12d.md'が作成される
```

注意点としては

- branch の名前に slug で許可されていない文字は入れられない("/"など)
- branch を切ってから記事を作成しないといけない
- branch の名前は 2 文字以上 40 文字以下にする(slug は 12 文字以上 50 文字以下なため)

などがありますが、後々のメンテナンスさや記事を作る時の気軽さを考えると、いい感じにできたと思います。
