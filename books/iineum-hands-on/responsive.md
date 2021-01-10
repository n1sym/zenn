---
title: "レスポンシブデザインにする"
---


ウィンドウ幅に合わせてレーン数を変更するようにしてみます。

~~~ts:src/Components/ImageTable.tsx
class ImageTable extends React.Component<typeImageTableProps, typeImageTableState> {
  constructor(props: typeImageTableProps) {
    super(props);
    this.state = {
      raneNum: window.innerWidth > 600 ? Math.floor(window.innerWidth / 300) : 2
    };
  }

  componentDidMount() {
    let queue: NodeJS.Timeout;
    window.addEventListener("resize", () => {
      clearTimeout(queue);
      queue = setTimeout(() => {
        const raneNum = window.innerWidth > 600 ? Math.floor(window.innerWidth / 300) : 2;
        this.setState({ raneNum: raneNum });
      }, 500);
    });
  }

...　略
~~~

`componentDidMount()`でイベントリスナーを設置しています。サイズ変更はかなりイベント発生の頻度が高い（ぐい～っと動かしている間、1/30秒刻みくらいで処理が発生する）ので処理を`setTimeout`でくくって制御しています。0.5秒間ウィンドウサイズが変わらなかったら、レーン数を更新します。


ウィンドウ幅が600px以下（スマホブラウザを想定）の場合は、レーン数は2で固定しています。