---
title: "モックを作る"
---

APIの準備が済んだので、Reactアプリに戻ります。

https://ja.reactjs.org/docs/thinking-in-react.html

まずは上記リンクを参考にしつつ、作るべきコンポーネントを考えます。

![](https://storage.googleapis.com/zenn-user-upload/telwcfyrvnc7zlh71hee895ne414)

4種類のコンポーネントが必要そうです。

MainTable（赤色）： このサイトの全体を含む
InputForm（青色）： ユーザ入力を受け付ける
ImageTable（緑色）： 画像を並べるところ（全体）
ImageList（黄色）： 画像を並べるところ（個別の列）
