---
title: "MagicaVoxelで作った3Dオブジェクトをサイトに表示させるまで　その２"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [threejs, javascript]
published: false
---

https://zenn.dev/hukurouo/articles/three-js-article-1

こちらの記事の続きです。前回作ったサンプルコードを用いて進めていきます。

https://github.com/hukurouo/voxel_sample/tree/feature/sample-basic

# TypeScriptの導入

まずはインストール。

~~~
npm i -D typescript ts-loader
~~~

`tsconfig.json`を新規に作成。

~~~json:tsconfig.json
{
  "compilerOptions": {
    "sourceMap": true,
    // TSはECMAScript 5に変換
    "target": "es5",
    // TSのモジュールはES Modulesとして出力
    "module": "es2015",
    // node_modules からライブラリを読み込む
    "moduleResolution": "node",
    "lib": [
      "es2020",
      "dom"
    ]
  }
}
~~~

`webpack.config.js`をts仕様に書き換える。

~~~js:webpack.config.js
module.exports = {
  // ローカル開発用環境を立ち上げる
  // 実行時にブラウザが自動的に localhost を開く
  devServer: {
    contentBase: "dist",
    open: true
  },
  entry: "./src/index.ts",
  // ファイルの出力設定
  output: {
    //  出力ファイルのディレクトリ名
    path: `${__dirname}/dist`,
    // 出力ファイル名
    filename: "main.js"
  },
  module: {
    rules: [
      {
        // 拡張子 .ts の場合
        test: /\.ts$/,
        // TypeScript をコンパイルする
        use: "ts-loader"
      }
    ]
  },
  // import 文で .ts ファイルを解決するため
  resolve: {
    extensions: [".ts", ".js"]
  },
};
~~~

`src/index.js` をリネームして、`src/index.ts`とします。型定義が効くようになりました。

`npm run dev` 中にtsファイルの方を書き換えても、自動でコンパイルが走りホットリロードされるので便利です。

# オブジェクト毎にページを作成する

~~~html:dist/index.html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Voxel Works</title>
</head>
<body>
  <h1>Voxel Works </h1>
    <ul>
      <li><a href='./spctr_switch/'>スペクトラちゃんとニンテンドースイッチ (20/11/14) </a></li>
      <li><a href='./spctr_room/'>スペクトラルーム (20/11/17) </a></li>
    </ul>
</body>
</html>
~~~

~~~html:src/template.html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><%= htmlWebpackPlugin.options.title %></title>
  </head>
  <body>
    <%= htmlWebpackPlugin.options.title %> :
    <a href='/'>TOP </a>
  </body>
</html>
~~~


# オブジェクトに影をつける

- a
  - a
    - aa
- a
  - aaaaaaa

# 簡易なローディング表示を追加する

