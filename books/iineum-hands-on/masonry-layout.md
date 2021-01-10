---
title: "Masonryレイアウトを作る"
---

このWebアプリの肝である、画像を並べるところ（Masonryレイアウト）を作っていきます。

> ImageTable（緑色）： 画像を並べるところ（全体）
ImageList（黄色）： 画像を並べるところ（個別の列）

![](https://storage.googleapis.com/zenn-user-upload/telwcfyrvnc7zlh71hee895ne414)

# Masonryレイアウトとは

高さの異なる画像を、隙間なく配置するデザインを Masonryレイアウト と呼ぶようです。Pinterest や tumblr など、画像を並べるサービスでよく使われているやつです。

このレイアウトの実装アプローチは色々とありそうですが、自分は以下のような形をとりました。

- 緑色の枠を横並びのflexコンテナ(親要素)にする
- ウィンドウの幅に合わせて、黄色の枠である flexアイテム(子要素)を複数作る
  - 以下、この黄色の枠はレーンと呼ぶ
- 各レーンに画像URLを持った配列を渡し、描画させる
  - このとき、各レーンを縦並びのflexコンテナとする
    - これで子要素である画像がレーン内で縦に並ぶ
  - 各レーンの高さが一定になるような配列を用意する
    - このために画像の高さ（正確には縦横比）を取得している

# 処理を書く

:::details src/Components/ImageTable.tsx
~~~ts:src/Components/ImageTable.tsx
import React from "react";
import ImageList from "./ImageList";

type typeImageTableProps = {
  images: typeItems;
};

type typeImageTableState = {
  raneNum: number;
};

type typeItems = {
  url: string[];
  height: number[];
  source: string[];
  max_id: string;
};

type typeRaneItems = {
  url: string;
  source: string;
};

class ImageTable extends React.Component<typeImageTableProps, typeImageTableState> {
  constructor(props: typeImageTableProps) {
    super(props);
    this.state = {
      raneNum: 3
    };
  }

  render() {
    return (
      <div className="flex m-1">
        {createRaneItems(this.state.raneNum, this.props.images).map((items: typeRaneItems[], index: number) => {
          return (
            <div key={index}>
              <ImageList raneItems={items} />
            </div>
          );
        })}
      </div>
    );
  }
}
export default ImageTable;

function createRaneItems(rane_num: number, items: typeItems): typeRaneItems[][] {
  const RaneItems: typeRaneItems[][] = Array(rane_num).fill([]).map(_i=>([]))
  const RaneHeights: number[] = Array(rane_num).fill(0);
  items.url.forEach((item: string, index: number) => {
    const minHeightIndex = searchMinHeightIndex(RaneHeights);
    RaneHeights[minHeightIndex] += items.height[index];
    RaneItems[minHeightIndex].push({ url: item, source: items.source[index] });
  });
  return RaneItems;
}

function searchMinHeightIndex(RaneHeights: number[]) {
  let minIndex = 0;
  let minHeight = 100000;
  RaneHeights.forEach((RaneHeight, index) => {
    if (minHeight > RaneHeight) {
      minIndex = index;
      minHeight = RaneHeight;
    }
  });
  return minIndex;
}
~~~
:::

:::details src/Components/ImageList.tsx
~~~ts:src/Components/ImageList.tsx
import React from "react";

type typeRaneItems = {
  url: string;
  source: string;
};
type typeImageListProps = {
  raneItems: typeRaneItems[];
};

class ImageList extends React.Component<typeImageListProps> {
  listItems = (items: typeRaneItems[]) => {
    return (
      <div className="flex flex-col">
        {items.map((item: typeRaneItems, index: number) => {
          return (
            <div key={index}>
              <div className="m-1 max-w-xs">
                <ListItem url={item.url} source={item.source} />
              </div>
            </div>
          );
        })}
      </div>
    );
  };

  render() {
    return <div>{this.listItems(this.props.raneItems)}</div>;
  }
}
export default ImageList;

function ListItem(props: typeRaneItems) {
  return (
    <a href={props.source} target="_blank" rel="noopener noreferrer">
      <img src={props.url} alt="" />
    </a>
  );
}
~~~
:::

:::details src/Components/MainTable.tsx
~~~ts:src/Components/MainTable.tsx
import React from "react";
import axios from "axios";
import InputForm from "./InputForm";
import ImageTable from './ImageTable'

...略

  render() {
    return (
      <div>
        <InputForm onSubmit={(screen_name: string) => this.handleSubmit(screen_name)}/>
        <ImageTable images={this.state.images}/>
        <div className="box h-64 text-center m-5 p-4 ...">
          {this.state.message}
        </div>
      </div>
    );
  }
}
export default MainTable;

...略
~~~
:::

ひとまず、レーン数は3で固定しています。

以下の箇所で、`raneNum`の値だけ`ImageList`コンポーネント（黄枠のレーン）を生成しています。

~~~ts:src/Components/ImageTable.tsx
return (
    <div className="flex m-1">
      {createRaneItems(this.state.raneNum, this.props.images).map((items: typeRaneItems[], index:number) => {
        return (
          <div key={index}>
            <ImageList raneItems={items} />
          </div>
        );
      })}
    </div>
  );
~~~

スクリーンネームを入力してツイートを取得してみます。

![](https://storage.googleapis.com/zenn-user-upload/hwtnq8vv38vtiu5ft5iz9xzdqfjy)

うまく並んでいます。これで原型ができました。