---
title: "Typescript+ESMでnpmパッケージ開発した備忘録"
emoji: "🗽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "npm"]
published: true
---
Typescript+ESMでnpmパッケージを作る方法が案外まとまってなかったので残しておく。マサカリ歓迎。ほぼ確実に間違い/非効率な点がある。jestでテストもやってる。基本yarnを使う。

モジュール化するときにCommonJS(requireのやつ)とESM(import/exportのやつ)とかがある。調べたところESMが業界標準で、nodejs>13.2.0とか __モダンな__ ブラウザではすでに対応してるらしいので、ESMで行く。Tree Shakeがしやすかったりするらしい。

## プロジェクトの初期化

```bash
YOUR_PACKAGE_NAME="<名前>"
mkdir $YOUR_PACKAGE_NAME && cd $YOUR_PACKAGE_NAME
yarn init -y
```

ESMはpackage.jsonに `"type": "module"`を指定しないといけないので追加しておく(`name`や`version`と同じ階層)。[参考](https://nodejs.org/api/packages.html#packages_determining_module_system)

## Typescriptの導入
```bash
yarn add -D typescript
yarn tsc --init
```

`tsconfig.json`はお好みだけど、ここら辺は指定しておくと良い。
```json
{
  "compilerOptions": {
    "target": "es6", // いい加減es6でよさそう
    "module": "esnext" ,　// ESMを使えるようにする。es2020でも可
    "declaration": true ,　//.d.tsを出力する
    "outDir": "./build" ,
    "rootDir": "./src" /,
    "esModuleInterop": true , // CJSとESMの相互運用を良い感じにしてくれるらしい
  },
  "exclude": ["**/*.test.*"],　// testはコンパイルしない
  "include": ["./src/**.*"] // ここにコードを置こう
}
```

es6にコンパイルする。nodejsでは6あたりから?、es6の機能が使えるようになったぽいので心配ない。ブラウザでもIE以外は対応してるし、そもそもバンドルするので問題ない。...とおもう。

## パッケージの中身を作成
```typescript:src/index.ts
export { IamExported } from "./module.js";
export const IamIndex = () => {
  console.log("I am exported in index.ts");
};
```

```typescript:src/module.ts
export const IamExported = (name: string) => {
  return `Hello, ${name}!! I am exported in module.ts!`;
};
```
index.tsに注意してほしい。`from "./module.js"`のように.jsを付けている。これは間違いではない。

:::details .jsの理由
`import {hoge} from "./module"`をTypescriptがコンパイルすると、手を加えずにそのまま出力される。しかしこれは正しいESMの記法ではない（多分）。本来のESMは拡張子が必要で、理想的にはTSに`import {hoge} from "./module.js"`とコンパイルしてほしい。しかしこれを実現するCompilerOptionを見つけられなかったので次善の策として.jsを付けている。.jsを付けても以前と同じように動くようだ。以下参照

https://fettblog.eu/typescript-and-es-modules/
:::

ここまできたら`yarn tsc`すると、`build/`にコンパイルされた.jsと.d.tsが出てくる。以下のような構成になるはず。
```
src/
  - index.ts
  - module.ts
  - module.test.ts

build/
  - index.js
  - index.d.ts
  - module.ts
  - module.d.ts
```

## publishの準備
以上でモジュール開発は終わりだが、npmにpublishするために追加で設定することがある。

まずpackage.json。最低でもこれは欲しい。
```json:package.json
{
  "main": "./build/index.js", // JSのエントリーポイント
  "types": "./build/index.d.ts", // TSのエントリーポイント
  "publishConfig": {
    "access": "public" // デフォルトはrestricted。無料アカウントならpublicが必須。
  },
  "type": "module" // ESMを使う（再掲）
}
```
ほかにはlicense、repositoryとかkeywordsも指定したほうが良い。以下参照。

https://docs.npmjs.com/cli/v7/configuring-npm/package-json
https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html

ここで`npm publish --dry-run`を実行すると、パッケージの概要が見れる。Tarball Contentsにはパッケージに同梱されるファイルが一覧で出てくる（多分）。
```
npm notice === Tarball Contents === 
npm notice 133B  babel.config.cjs 　
npm notice 122B  build/index.js    
npm notice 101B  build/module.js   
npm notice 503B  package.json      
npm notice 900B  tsconfig.json     
npm notice 6.5kB jest.config.mjs   
npm notice 83B   build/index.d.ts  
npm notice 119B  src/index.ts      
npm notice 60B   build/module.d.ts 
npm notice 149B  src/module.test.ts
npm notice 107B  src/module.ts      
```

babel.config.cjsやjest.config.mjs, src/以下のファイルは同梱しても意味ないので省きたい。そういう時は`.npmignore`を使う。

```ignore:.npmignore
src
*config.*
```

これを作ってもう一度`npm publish --dry-run`をする。
```
npm notice === Tarball Contents === 
npm notice 122B build/index.js   
npm notice 101B build/module.js  
npm notice 503B package.json     
npm notice 83B  build/index.d.ts 
npm notice 60B  build/module.d.ts
```

必要なファイルだけ同梱していることがわかる。

## 動作確認
`yarn link`を使うとローカルでテストできる。パッケージとは別のディレクトリで動作確認をする。

```bash
yarn link
cd ..
mkdir package-test && cd package-test
yarn init -y　# "type": "module"を追加しておく
yarn link $YOUR_PACKAGE_NAME
```

`package-test/node_modules`をみると、作ったパッケージへシンボリックリンクが貼られているのがわかる。だから次のように簡単にモジュールの動作確認ができる。package.jsonに`"type":"module"`を忘れないように。

```js
import { IamExported, IamIndex } from "$YOUR_PACKAGE_NAME"
console.log(IamExported("arark")); // Hello, arark!! I am exported in module.ts!
IamIndex(); // I am exported in index.ts
```
## npmへpublish
npmのアカウントを作る。メール認証しないとpublishが403ではじかれるので注意。

```bash
yarn login
yarn publish
```

ここは良い感じにやって（飽きた）。

package.jsonにこう書いておくとpublishとかの前にビルドしてくれる。
```json
"scripts": {
  "prepare": "tsc"
 },
```
## （付録）jestの導入

テストスクリプトを書く。
```typescript:src/module.test.ts
import { IamExported } from "./module";
test("IamExported returns greeting", () => {
  expect(IamExported("arark")).toContain("Hello, arark!!");
});
```

jestの設定をする。
```bash
yarn add jest @types/jest -D
yarn run jest --init
✔ Would you like to use Jest when running "test" script in "package.json"? … yes
✔ Would you like to use Typescript for the configuration file? … no
✔ Choose the test environment that will be used for testing › node
```

これで`yarn test`やるとimportがわかんねえよ！と怒られる(jestがESMにネイティブ対応してないからbabelを咬ませなきゃいけないっぽい？)。
よくわかんないけど[ドキュメント](https://jestjs.io/docs/ja/getting-started#typescript-%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B)に沿うと解決できる。具体的には:

```bash
yarn add -D babel-jest @babel/core @babel/preset-env  @babel/preset-typescript
echo 'module.exports = { presets: [["@babel/preset-env", { targets: { node: "current" } }], "@babel/preset-typescript", ],};' > babel.config.cjs
```

でbabelを設定する。わざわざ.cjsにしたのは、configだけCommonJSを使いたいから。.jsではpackage.jsonの`"type": "module"`からESMと解釈されるのでだめ。

これでjestが動くはず。
```bash
$ yarn test
yarn run v1.22.5
$ jest
 PASS  src/module.test.ts
  ✓ IamExported returns greeting (2 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.8 s
Ran all test suites.
Done in 1.35s.
```
