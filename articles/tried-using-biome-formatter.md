---
title: "社内ツールでBiome formatterを試してみた"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "typescript", "biome"]
published: true
publication_name: moneyforward
published_at: 2023-12-21 09:00
---

この記事は、[Money Forward Engineering 2 Advent Calendar 2023](https://adventar.org/calendars/9263) 21日目の投稿です。

---

## はじめに

フロントエンド開発の世界では新しいツールやライブラリが絶えず登場しています。この記事では、最近特に注目を集めているBiomeのformatter機能（以下、Biome formatter）を社内ツールのプロジェクトで試してみたので、その内容を共有します。

## Biomeとは

Biomeは、Web開発に必要な様々なツールチェーンを開発するプロジェクトです。現在はJavaScriptやTypeScript、JSONをターゲットにlinterやformatterといった機能を提供しています。
Biomeの歴史や特徴、詳細な機能についてはこの記事では触れませんので、公式ドキュメントや以下の記事を参照してください。

https://biomejs.dev/
https://biomejs.dev/blog/annoucing-biome/

## Biome formatterについて

Biome formatterは、Prettier challengeを勝ち取ったことで最近注目を集めています。これはBiome formatterがPrettierと十分な互換性を持っていることを示しています。

https://biomejs.dev/blog/biome-wins-prettier-challenge/
https://zenn.dev/sosukesuzuki/articles/e1e47e2a760e9d

## Biome formatterを試してみた

Prettierを使っているプロジェクトであれば、Biome formatterを試してそれらを比較してみることは簡単でしょう。というわけで、今回は実験場として社内ツールのプロジェクトを利用し、Biome formatterを試してみました。

:::message
Biomeのlinter機能は現状プラグイン機能を提供しておらず、通常多くのプラグインを駆使してセットアップするESLintの置き換えを試すにはまだギャップがあります。そのため、今回はlinter機能については試していません。
:::

### 社内ツールのプロジェクトの概要

今回対象とした社内ツールのプロジェクトは、以下の技術スタックで構成されています。

- React
- TypeScript
- Next.js
- Styled Components
- ESLint / Prettier

### Biomeをセットアップする

まずは、Biomeをインストールします。

```bash
yarn add -D @biomejs/biome
```

次に、Biomeの設定ファイルを作成します。設定ファイルはBiome CLIの`init`コマンドで生成されますが、`node_modules/.bin`配下に置かれる実行ファイル名はパッケージ名の`@biomejs/biome`ではなく`biome`であることに注意します。
`npx`や`yarn/pnpm dlx`コマンドを利用する場合は、パッケージ名の`@biomejs/biome`を指定しましょう。

```bash
yarn biome init
# or
yarn dlx @biomejs/biome init
```

これでデフォルトの設定ファイル`biome.json`が生成されました。あとは、この設定ファイルを編集していきます。

下記が今回使うBiomeの設定ファイルです。
formatterの設定は、既存のPrettierの設定とBiomeの設定オプションを見比べながらほぼ同じ設定になるように調整しました。
また、linter機能やAnalyzerとして提供されているimport文のソート機能は今回利用しないため、無効化しています。

```json
{
  "$schema": "https://biomejs.dev/schemas/1.4.1/schema.json",
  "organizeImports": {
    "enabled": false
  },
  "linter": {
    "enabled": false
  },
  "formatter": {
    "enabled": true,
    "ignore": ["node_modules/*", ".next/*", "public/*", "src/__generated__/*"],
    "indentStyle": "space",
  },
  "javascript": {
    "formatter": {
      "enabled": true,
      "quoteStyle": "single",
      "jsxQuoteStyle": "double",
      "trailingComma": "es5"
    }
  }
}
```

formatterの設定オプションはPrettierの設定オプションの多くをサポートしており、Prettierと同じように既存の設定を引き継ぐことができます。
詳しくは以下のドキュメントを参照してください。

https://biomejs.dev/reference/configuration/

### Biome formatterを実行する

設定ファイルの準備ができたので、Biome formatterを実行してみます。

```bash
yarn biome format --write .
# 省略
Formatted 204 file(s) in 78ms
```

200ファイル以上を100ms以内にフォーマットしました。Prettierの場合は3秒ほどかかっていたので、驚くべきパフォーマンスを発揮していることがわかります。

:::message
私の試したプロジェクトの規模感では、Prettierのcache機能を利用するとBiomeの実行速度と同程度になります。
https://prettier.io/docs/en/cli.html#--cache

すべてのプロジェクトで同様の結果になるとは限りませんが、このようにcache機能を活用することでPrettier実行のパフォーマンスを大幅に改善できる可能性があります。
:::

フォーマット結果を見てみると、ネストされた長い三項演算子を使ったコードなどいくつかの部分でフォーマットがPrettierと異なることがあったりしましたが、ほとんどのファイルで同じフォーマット結果になっていました。正しく設定できれば、Prettierからの移行も十分に可能だと思います。

## まとめ

今回は社内ツールのプロジェクトを土台にしてBiome formatterを少し試してみた経験を共有しました。Biome formatterはPrettierと互換性が高く、パフォーマンスも優れているため、今後もWeb開発におけるformatterの選択肢の一つとして注目されることでしょう。

これを読んだ方が、Biome formatterを試してみるきっかけになれば幸いです。
