---
title: "個人ブログのパッケージマネージャーをYarnからpnpmに移行した作業ログ"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "yarn", "pnpm"]
published: true
published_at: 2023-01-05 09:00
---

新年早々Yarnを利用していた個人ブログのプロジェクトをpnpmに移行したので、その際の作業ログを記事にします。

# 個人ブログの技術構成

私の個人ブログは以下のような技術構成を取っています。

- パッケージマネージャー: Yarn v1
- フレームワーク: Next.js
- UIライブラリ: Chakra UI
- CI: GitHub Actions
- ホスティング: Vercel

# pnpm

pnpmは速さや効率性、厳格さを特徴に掲げるNode.js向けのパッケージマネージャーです。

https://pnpm.io/

本記事ではその特徴の詳細について記載しないため、下記のような解説記事を参考ください。

https://zenn.dev/azukiazusa/articles/pnpm-feature
https://zenn.dev/wakamsha/articles/construct-monorepo-with-pnpm

## pnpmへの移行動機

npmやYarnは使った経験がありましたがpnpmはなかったので、個人ブログのリポジトリを使って技術検証してみようと思ったのが最初の動機です。

また、最近プライベートで利用しているMacのストレージがカツカツになってきたので、ディスク効率のいいpnpmに全面的に移行してローカルマシンの容量節約を図る意図もあります。

# 移行

## pnpmの有効化

さっそくpnpmを利用できるようにします。
現在利用しているNode.jsのバージョンはCorepackが同梱されているので、Corepack経由で利用します。

```sh
$ node -v
v18.12.1

$ corepack enable
$ corepack prepare pnpm@latest --activate
```

私はasdfを利用してNode.jsのバージョン管理しているため、Corepack経由でインストールしたpnpmを利用するためにreshimする必要がありました。

```sh
$ asdf reshim nodejs
```

https://zenn.dev/matunnkazumi/articles/node_corepack_in_asdf

さらに、`package.json`に`packageManager`フィールドを追加してプロジェクトで利用するパッケージマネージャーのバージョンを指定します。

```json:package.json
{
  "packageManager": "pnpm@7.21.0"
}
```

以上でpnpmを有効化できました。

## `pnpm import`

pnpmには`pnpm import`というコマンドが用意されています。これはnpmやYarnなど他のパッケージマネージャーのlockfileから`pnpm-lock.yaml`を生成するコマンドです。Yarnにも同等の機能を有する`yarn import`コマンドが存在します。

https://pnpm.io/cli/import

純粋にlockfileを再生成すると、依存するパッケージのバージョンが意図せず上がってしまうことがあります。既存のlockfileから新たなlockfileを生成することで、各パッケージのバージョンを移行前と変えることなくパッケージマネージャーを移行できることが期待されます。

:::message
実際に移行前とまったく同じバージョンで生成できているのかの検証は行っていません。
:::

`pnpm import`を実行すると、自動で`yarn.lock`を検出して`pnpm-lock.yaml`を生成してくれました。

```sh
$ pnpm import
```

## `pnpm install`

lockfileの移行ができたので、`pnpm install`を実行して`node_modules`を作り直します。

```sh
$ rm -rf node_modules
$ pnpm install
```

## `save-exact=true`にする

個人ブログのリポジトリではRenovateを使ってパッケージを更新しています。

Renovateを利用する上では`package.json`におけるパッケージバージョンのSemVer range指定（`^1.0.0`や`~1.0.0`）は利用しないことが多いです。そのため、`pnpm add`する際に自動でバージョンが固定（pin）されるようにします。

pnpmのドキュメントには記載されていませんが、pnpmは`.npmrc`の`save-exact`値の設定を尊重して動作を変えてくれます。今回はこちらを利用しました。

https://docs.npmjs.com/cli/v9/using-npm/config#save-exact

Yarnを利用している間は`.yarnrc`の`save-prefix ''`という設定でバージョンが固定されるようにしていたので、このファイルを削除した上で下記コマンドを実行して`.npmrc`を作成しました。

https://pnpm.io/cli/config

```sh
$ rm .yarnrc
$ pnpm config set save-exact true
```

```ini:.npmrc
# このようなファイルが生成されます
save-exact=true
```

:::message
Yarnにおいても上記のような`.npmrc`ファイルの設定が尊重されるようになっています。移行前に`.npmrc`を使ってバージョンを固定していた場合、この設定は不要です。

参考: https://github.com/yarnpkg/yarn/issues/1088
:::

## CIの修正

個人ブログのリポジトリでは、GitHub ActionsでESLintやNext.jsのビルドの実行などを行っています。パッケージのインストールはYarnに依存しているため、当然修正が必要になります。

pnpmのセットアップを行うための公式のActionが用意されているためこちらを利用します。

https://github.com/pnpm/action-setup
https://pnpm.io/continuous-integration#github-actions

```diff yaml:.github/workflows/linter.yml
name: lint
on:
  push:
    branches:
      - main
  pull_request:
jobs:
  eslint:
    name: eslint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
+      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v3
        with:
          node-version-file: '.tool-versions'
-          cache: 'yarn'
-      - run: yarn --frozen-lockfile
+          cache: 'pnpm'
+      - run: pnpm install --frozen-lockfile
      - name: run eslint
-        run: yarn lint
+        run: pnpm lint
```

:::message
`package.json`の`packageManager`フィールドを設定しているため、`pnpm/action-setup`の`version` inputを指定する必要はありません（自動でバージョンを検出してくれます）。

参考: https://github.com/pnpm/action-setup#version
:::

:::message
`actions/setup-node`のキャッシュ機能はpnpmもサポートしています。

参考: https://github.com/actions/setup-node/blob/main/docs/advanced-usage.md#caching-packages-data
:::

## Yarnに依存していた箇所の修正

その他、コード内でYarnに依存していた箇所を修正しました。

今回のリポジトリではhusky + lint-stagedを使ってpre-commit時にESLintやPrettierを実行しています。
lint-stagedを動かすために`yarn lint-staged`というコマンドを実行するようにしていたため、この部分もpnpmに変更しました。

```diff sh:.husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

- yarn lint-staged
+ pnpm lint-staged
```

## パッケージimportの修正

pnpmは、デフォルトでは`package.json`に書かれていない暗黙的な依存パッケージをアプリケーションコードから参照できません。これはnpmやYarnが行っているようなパッケージの巻き上げ（hoisting）を伴うフラットな`node_modules`の構造をpnpmが採用していないためです。

https://pnpm.io/ja/blog/2020/05/27/flat-node-modules-is-not-the-only-way

npm/Yarnを使っていたときにhoistingされた暗黙的な依存パッケージをアプリケーションコードから呼び出していた場合、pnpmへ移行すると正常に参照できなくなります。

今回の個人ブログではUIライブラリとしてChakra UIを利用しています。
Chakra UIでは各種UIコンポーネントは`@chakra-ui/button`や`@chakra-ui/layout`といった小さなライブラリに分割されており、`@chakra-ui/react`をインストールすることで分割されたライブラリを暗黙的に利用しています。`@chakra-ui/react`では単にこれらの分割されたライブラリのコンポーネントや型をre-exportしているだけです。

https://github.com/chakra-ui/chakra-ui/blob/main/packages/components/react/src/index.ts

VSCodeなどのエディタやIDEで自動import機能を使っていると、意図せず`@chakra-ui/layout`などからコンポーネントを直接importしてしまうケースがあります。私のリポジトリではこのケースがいくつか存在しておりビルドが失敗していたため、`@chakra-ui/react`からコンポーネントをimportするように修正しました。

```diff tsx:components/Foo.tsx
- import { Heading, Text } from '@chakra-ui/layout';
+ import { Heading, Text } from '@chakra-ui/react';
```

## デプロイ

個人ブログのデプロイ先にはVercelを利用し、Vercelの提供するGitHub連携でデプロイされています。パッケージインストールコマンドは自動で検出されるプロジェクト内のlockfileによって変わります。pnpmもサポートされているため、特に設定の変更なくデプロイできました。

https://vercel.com/docs/concepts/deployments/configure-a-build#install-command

一方、Corepackは実験的ツールであるため、VercelのデプロイフローにおいてデフォルトではCorepackを利用することはありません。`ENABLE_EXPERIMENTAL_COREPACK`環境変数を`1`に設定することでCorepackを有効化できます。
試しにCorepackを有効化してみましたが、問題なくデプロイできました。

https://vercel.com/docs/concepts/deployments/configure-a-build#corepack

# まとめ

大きな落とし穴もなくYarnからpnpmへの移行を完了できました。

移行作業や実際の開発作業を少し行ってみましたが特に問題もなかったため、他のリポジトリも移行していくつもりです。
