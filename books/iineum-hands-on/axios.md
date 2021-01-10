---
title: "axiosでAPIと通信する"
---



https://github.com/axios/axios

HTTPクライアントとしてaxiosをインストールします。

~~~sh
$ npm install axios
~~~

APIとの通信についての処理はMainTableコンポーネントに書いていきます。

:::details src/Components/MainTable.tsx
~~~ts:src/Components/MainTable.tsx
import React from "react";
import axios from "axios";
import InputForm from "./InputForm";

type typeImageTableState = {
  images: typeImages;
  message: string;
  screen_name: string;
};

type typeImages = {
  url: string[];
  height: string[];
  source: string[];
  max_id: string;
};

class MainTable extends React.Component<{}, typeImageTableState> {
  constructor(props: {}) {
    super(props);
    this.state = {
      images: {
        url: [],
        height: [],
        source: [],
        max_id: "",
      },
      message: "",
      screen_name: "",
    };
  }

  handleSubmit = (screen_name: string) => {
    twitterAPI(screen_name, this.state.images.max_id)
    .then((res) => {
      this.setIineImages(res);
    })
    .catch(() => {
      this.setState({
        message: "取得に失敗しました。データが空か、スクリーンネームが間違っているかもしれません。",
      });
    });
  }

  setIineImages = (results: any) => {
    this.setState({images: results, message: "done"})
    console.log(this.state.images)
  };
  
  render() {
    return (
      <div>
        <InputForm onSubmit={(screen_name: string) => this.handleSubmit(screen_name)}/>
        <div className="box h-64 text-center m-5 p-4 ...">
          {this.state.message}
        </div>
      </div>
    );
  }
}
export default MainTable;

function twitterAPI(screen_name: string, max_id: string) {
  let endpoint = `${process.env.REACT_APP_API_ENDPOINT_URL}/fav?name=${screen_name}&maxid=${max_id}`
  return new Promise((resolve, reject) => {
    axios.get(endpoint)
      .then((res) => {
        resolve(res.data);
      })
      .catch((err) => {
        reject(err);
      });
  });
}
~~~
:::

APIのエンドポイントをハードコーディングするのは良くないので、環境変数として読み込んでいます。

~~~ts:src/Components/MainTable.tsx
  let endpoint = `${process.env.REACT_APP_API_ENDPOINT_URL}/fav?name=${screen_name}&maxid=${max_id}`
~~~

このアプリは Create React App で作っているので、環境変数は `.env`ファイルをルートディレクトリに置くだけで設定されます。

ただし変数名は`REACT_APP_`をプレフィックスにする必要があります。また、`.env`設定を反映させるために一旦サーバーを再起動する必要があります。

~~~env:.env
REACT_APP_API_ENDPOINT_URL=https://**********.execute-api.ap-northeast-1.amazonaws.com/production
~~~

:::message alert
`.gitignore` に `.env` を追記することを忘れずに！
:::


参考：
https://create-react-app.dev/docs/adding-custom-environment-variables/#adding-development-environment-variables-in-env



試しにフォームに任意のユーザーのスクリーンネームを入力してみます。

![](https://storage.googleapis.com/zenn-user-upload/bsvfxmu2xdt90np3vzhuddhzipxn)

ツイートデータが取得できています。また、ボタンを押すたびに古いツイートが取得されていることも分かります。

存在しないスクリーンネームなどを入力すると、エラーメッセージが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/um8mzfshwzlezw7778hdp4gbt01s)
