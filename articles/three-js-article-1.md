---
title: "MagicalVoxelで作った3Dオブジェクトをサイトに表示させるまで"
emoji: "🧱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [threejs, javascript]
published: true
---

# はじめに

windows環境です。
初学者なので不備などありましたらご指摘ください。


# MagicalVoxelで3Dオブジェクトを作る

ニンテンドースイッチを作ってみました。これをブラウザで表示させたい。

![](https://storage.googleapis.com/zenn-user-upload/oiod4oraqk5pkhua7b21dtcdo4cy)

objファイルにエクスポートしておきます。

![](https://storage.googleapis.com/zenn-user-upload/sev7sxs8llo8p0figryk8qego9lz)

# three.js の開発環境を作る

https://threejs.org/

> three.jsは、ウェブブラウザ上でリアルタイムレンダリングによる3次元コンピュータグラフィックスを描画する、クロスブラウザ対応の軽量なJavaScriptライブラリ及びアプリケーションプログラミングインタフェースである。[wikipedia](https://ja.wikipedia.org/wiki/Three.js)

この three.js というライブラリを用いて、レンダリングを試みます。

[最新版で学ぶwebpack 5入門 JavaScriptのモジュールバンドラ](https://ics.media/entry/12140/)

こちらを参考に three.js の開発環境を作っていきます。

プロジェクトのルートディレクトリに移動して、webpack一式とthree.jsをインストールします。
~~~
$ cd c:\code\three-sample

$ npm init -y

$ npm install -D webpack webpack-cli webpack-dev-server babel-loader @babel/core  @babel/preset-env

$ npm install three
~~~

`package.json` が生成されます。便利のため、`scripts`にビルドコマンド等を追記しておきます。

```json:package.json
{
  "scripts": {
    "build": "webpack",
    "dev": "webpack serve"
  },
  "dependencies": {
    "three": "^0.121.1"
  },
  "devDependencies": {
    "@babel/core": "^7.12.3",
    "@babel/preset-env": "^7.12.1",
    "babel-loader": "^8.2.1",
    "webpack": "^5.4.0",
    "webpack-cli": "^4.2.0",
    "webpack-dev-server": "^3.11.0"
  },
  "private": true
}
```

簡単なファイルを作って試運転してみます。`webpack.config.js`を作成して、

```js:webpack.config.js
module.exports = {
  // モード値を production に設定すると最適化された状態で、
  // development に設定するとソースマップ有効でJSファイルが出力される
  mode: "development",

  // ローカル開発用環境を立ち上げる
  // 実行時にブラウザが自動的に localhost を開く
  devServer: {
    contentBase: "dist",
    open: true
  }
};
```

srcフォルダ下に index.js を作ります。現時点でのディレクトリ構成はこのような感じ。

```js:index.js
alert("hello world");
```


```
.
├── node_modules
├── src
│   └── index.js
├── package-lock.json
├── package.json
└── webpack.config.js
```



~~~
$ npm run build
~~~

ビルドコマンドを入力すると、distフォルダが生えてきます。

distフォルダ内に `main.js` が生成されています。

同フォルダ内に `index.html` を作って、動作を確認してみます。

```html:index.html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
  </style>
</head>
<body>
  <script src="main.js"></script>
  <div>hello :D</div>
</body>
</html>
```


```
.
├── dist
│   ├── index.html
│   └── main.js
├── node_modules
├── src
│   └── index.js
├── package-lock.json
├── package.json
└── webpack.config.js
```

~~~
$ npm run dev
~~~

`http://localhost:8080/` が立ち上がり、動作確認ができました。

index.js を書き換えると、自動で反映してくれています。

# three.js の動作を確認する

index.js を書き換えて、three.js を動かしてみます。

https://threejs.org/docs/index.html#manual/en/introduction/Creating-a-scene

公式のサンプルコードからお借りします。


```js:index.js
import * as THREE from "three";

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera( 75, window.innerWidth / window.innerHeight, 0.1, 1000 );

const renderer = new THREE.WebGLRenderer();
renderer.setSize( window.innerWidth, window.innerHeight );
document.body.appendChild( renderer.domElement );

const geometry = new THREE.BoxGeometry();
const material = new THREE.MeshBasicMaterial( { color: 0x00ff00 } );
const cube = new THREE.Mesh( geometry, material );
scene.add( cube );

camera.position.z = 5;

const animate = function () {
	requestAnimationFrame( animate );
	cube.rotation.x += 0.01;
	cube.rotation.y += 0.01;
	renderer.render( scene, camera );
};
animate();
```

真っ黒な空間で緑色の箱がくるくる回っていますね。

# three.js で3Dオブジェクトファイルを読み込む

用意しておいたオブジェクトファイルを格納するためのmodelsフォルダをdistに用意します。

```
.
├── dist
│   ├── models
│   │   ├── switch.mtl
│   │   ├── switch.obj
│   │   └── switch.png
│   ├── index.html
│   └── main.js
```


サンプルコードにオブジェクトを読み込ませる記述を書き加えていきます。

```js:index.js
import * as THREE from "three";
// オブジェクトをロードするための Loader をimportしておきます
import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader.js';
import { MTLLoader } from 'three/examples/jsm/loaders/MTLLoader.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera( 75, window.innerWidth / window.innerHeight, 0.1, 1000 );

const renderer = new THREE.WebGLRenderer();
renderer.setSize( window.innerWidth, window.innerHeight );
document.body.appendChild( renderer.domElement );

// オブジェクトを回転させるときに参照したいため、ここで変数を宣言します
var object_switch = null;

// オブジェクトを読み込む
const mtlLoader = new MTLLoader();
mtlLoader.setPath('models/');
mtlLoader.load('switch.mtl', (materials) => {
  materials.preload();
  const objLoader = new OBJLoader();
  objLoader.setMaterials(materials);
  objLoader.setPath('models/');
  objLoader.load('switch.obj', (object) => {
    const mesh = object;
    object_switch = object;
    scene.add(mesh);
  });
})

camera.position.z = 5;

// オブジェクトを照らすために環境光源を追加する
const light = new THREE.AmbientLight(0xFFFFFF, 1.0);
scene.add(light);

const animate = function () {
  requestAnimationFrame( animate );
  // そのまま書くとオブジェクトが読み込まれる前に動いてしまうので、ifで括っておく
  if (object_switch){
    object_switch.rotation.x += 0.01;
    object_switch.rotation.y += 0.01;
  }
  renderer.render( scene, camera );
};
animate();
```

[サンプル](https://stoic-tereshkova-0666e1.netlify.app/)

真っ黒な空間でスイッチがくるくる回っています。


# ビルド環境を作る

本番環境用のファイルを生成するためのビルド環境を作っていきます。

まずは`webpack.config.js`を書き換えます。

参考：
https://ics.media/entry/16028/#webpack-babel-three

```js:webpack.config.js
module.exports = {
  // ローカル開発用環境を立ち上げる
  // 実行時にブラウザが自動的に localhost を開く
  devServer: {
    contentBase: "dist",
    open: true
  },
  entry: "./src/index.js",
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
        // 拡張子 .js の場合
        test: /\.js$/,
        use: [
          {
            // Babel を利用する
            loader: "babel-loader",
            // Babel のオプションを指定する
            options: {
              presets: [
                // プリセットを指定することで、ES2020 を ES5 に変換
                "@babel/preset-env"
              ]
            }
          }
        ]
      }
    ]
  },
};
```

`package.json` を編集して、mode はコマンドで指定するようにします。

```json:package.json
  "scripts": {
    "build": "webpack --mode production",
    "dev": "webpack serve --mode development --watch"
  },
```

これでビルドしてあげると、より圧縮されたmain.jsが生成されます。

~~~
npm run build
~~~

# netlifyでサイトを公開する

github でリポジトリを作って、それと連携するだけでサイトが公開できます。

Build command は `npm run build`
Publish directory は `dist`

を指定しましょう。

https://stoic-tereshkova-0666e1.netlify.app/

無事に公開されました。

# おわりに

こちらサンプルコードです。
https://github.com/hukurouo/voxel_sample

