---
title: "ページタイトルを変える"
---


デフォルトだと`React App`で不格好なので、ページタイトルを変えます。

https://github.com/staylor/react-helmet-async

~~~sh
npm install react-helmet-async
~~~

~~~ts:src/App.tsx
import MainTable from "./Components/MainTable";
import { Helmet, HelmetProvider } from "react-helmet-async";

function App() {
  return (
    <HelmetProvider>
      <Helmet>
        <title>iine-app-handson</title>
      </Helmet>
      <div className="bg-blue-50 min-h-screen">
        <div className="container mx-auto">
          <header className="flex justify-center items-center text-3xl h-32 mx-5">
            いいねした画像を並べるサイト
          </header>
          <div className="flex justify-center">
            <MainTable />
          </div>
        </div>
      </div>
    </HelmetProvider>
  );
}

export default App;
~~~

以下の箇所で任意のタイトルに変更できます。

~~~ts:src/App.tsx
      <Helmet>
        <title>iine-app-handson</title>
      </Helmet>
~~~