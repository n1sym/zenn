---
title: "Reactのセットアップ"
---

# Create React App でプロジェクトを作る

まずは、[Create React App](https://github.com/facebook/create-react-app) を使ってReactとTypeScriptのプロジェクトをセットアップします。


~~~sh
$ npm i -g create-react-app
~~~


~~~sh
$ npx create-react-app iine-app-handson --template typescript
~~~

セットアップには数分かかるかも。

参考：
https://typescript-jp.gitbook.io/deep-dive/browser#2-create-react-appwosuru

:::message
VSCode内のTypeScriptのバージョンが古いと警告が出ます。v4.1以上にアップデートしましょう。

https://code.visualstudio.com/docs/typescript/typescript-compiling#_using-newer-typescript-versions
:::

# tailwindcss の導入

CSSライブラリとしてtailwindcssを使います。細かいカスタマイズが簡単なのが魅力です。

[公式ドキュメント](https://tailwindcss.com/docs/guides/create-react-app#setting-up-tailwind-css)に従ってインストールします。書いてあることをそのままコピペすれば大丈夫なので、ここは省略します。

参考：
https://tailwindcss.com/docs/guides/create-react-app#setting-up-tailwind-css

# 起動テスト

初期ファイルの`src/App.tsx`を書き換えます。

CSSはtailwindcssで設定しています。初めて使う方は、[リファレンス](https://tailwindcss.com/docs)を眺めつつ `className` をいじって遊んでみるとよいかもしれません。


~~~ts:src/App.tsx
function App() {
  return (
    <div className="container mx-auto">
      <header className="flex justify-center items-center text-3xl h-32 mx-5">
        hello :D
      </header>
      <div className="flex justify-center">
        content
      </div>
    </div>
  );
}

export default App;
~~~

`npm run start`でローカル起動してみます。以下のような簡単なページが表示されたらOKです。

![](https://storage.googleapis.com/zenn-user-upload/fnhxs6za57jdwdux5y3nsxv2hgk9)

`src/App.css` と `src/logo.svg` は今後使用しないので消去しても問題ありません。
