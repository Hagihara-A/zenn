---
title: "Typescript+ESMでnpmパッケージ開発した備忘録"
emoji: "🗽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "npm"]
published: false
---
Typescript+ESMでnpmパッケージを作る方法が案外まとまってなかったので残しておく。マサカリ歓迎。jestでテストもやってる。基本yarnを使う。

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

見てのとおりes6にコンパイルする。nodejsでは6あたりから?、es6の機能が使えるようになったぽいので心配ない。ブラウザでもIE以外は対応してるし、そもそもバンドルするので問題ない。...とおもう。

## パッケージの中身を作成
```typescript:src/index.ts
export { IamExported } from "./module";
export const IamIndex = () => {
  console.log("I am index.ts");
};
```

```typescript:src/module.ts
export const IamExported = (name: string) => {
  console.log(`Hello, ${name}!! I am exported in module.ts!`);
};
```

## jestの導入

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

これでjestは動くはず。
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

これでjestは終わり。


## package.jsonを整える
ここまできたら`yarn tsc`すると、`build/`にコンパイルされた.jsと.d.tsが出てくる。`module.test.js`と`module.test.d.ts`も出力されてたらそれは間違い。おそらくtsconfigのexcludeが違ってる。
