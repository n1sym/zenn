---
title: "Next.js のチュートリアルで作ったブログに5分でいいねボタンを実装する"
emoji: "👍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

# はじめに

いいねボタンが押されたら Discord のチャンネルに通知がいく仕組みを作ります。

# ソースコードの準備

Next.js のチュートリアルが完了した状態から話を進めます。

https://nextjs.org/learn/foundations/about-nextjs

以下を実行すれば手っ取り早く環境を再現できます。

~~~
npx create-next-app nextjs-blog-iine --use-npm --example "https://github.com/vercel/next-learn/tree/master/basics/typescript-final"
~~~

# Discord で ウェブフックURLを取得する

サーバー設定 > 連携サービス > ウェブフック > 新しいウェブフック > ウェブフックURLをコピー で取得できます。

取得したウェブフックURLは環境変数にセットしておきます。

~~~: .local.env
NEXT_PUBLIC_DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/********
~~~

https://nextjs.org/docs/basic-features/environment-variables

# 実装

~~~ts:lib/discord.ts
const webHookUrl = process.env.NEXT_PUBLIC_DISCORD_WEBHOOK_URL

export function postMessage(title: string) {
  const data = { "username": "blog-notify", "content": title + "がいいねされました" }
  fetch(webHookUrl, {
    method: "POST",
    mode: "cors",
    headers: {
      'Content-type': 'application/json'
    },
    body: JSON.stringify(data),
  }).catch(error => console.error(error))
}
~~~

~~~tsx:components/IineButton.tsx
import { useState } from 'react'
import { postMessage } from '../lib/discord'

export function IineButton({ title }: { title: string }) {
  const [isDisplay, setIsDisplay] = useState(false)

  function postIine(title: string) {
    postMessage(title)
    toggleDisplay()
  }

  function toggleDisplay() {
    setIsDisplay(!isDisplay)
  }

  return (
    <>
      <button disabled={isDisplay ? true : false} onClick={() => postIine(title)}>
        👍
      </button>
      <span style={{ display: isDisplay ? '' : 'none' }}> {"<"} thank you ! </span>
    </>
  )
} 
~~~

~~~diff tsx:pages/posts/[id].tsx
  import Layout from '../../components/layout'
  import { getAllPostIds, getPostData } from '../../lib/posts'
  import Head from 'next/head'
  import Date from '../../components/date'
  import utilStyles from '../../styles/utils.module.css'
  import { GetStaticProps, GetStaticPaths } from 'next'
+ import { IineButton } from '../../components/IineButton'
  
  export default function Post({
    postData
  }: {
    postData: {
      title: string
      date: string
      contentHtml: string
    }
  }) {
    return (
      <Layout>
        <Head>
          <title>{postData.title}</title>
        </Head>
        <article>
          <h1 className={utilStyles.headingXl}>{postData.title}</h1>
          <div className={utilStyles.lightText}>
            <Date dateString={postData.date} />
          </div>
          <div dangerouslySetInnerHTML={{ __html: postData.contentHtml }} />
+         <IineButton title={postData.title}/>
        </article>
      </Layout>
    )
  }
~~~

:::details おまけ：tailwindcss でボタンを装飾する場合

[自分のブログ](https://hukurouo.com/articles) では `tailwindcss` を使っているので、以下のように実装しています。

![](https://storage.googleapis.com/zenn-user-upload/324ca5f3fe62-20220604.png)

~~~tsx
    <>
      <button className="bg-transparent text-blue-700 border py-1 px-4 ml-4 mr-2 rounded-full hover:bg-gray-100" disabled={isDisplay ? true : false } onClick={() => postIine(title)}>
        👍
      </button>
      <span style={{ display: isDisplay ? '' : 'none' }}> {"<"} thank you ! </span>
    </>
~~~

:::



# 動作確認

`npm run dev` で環境を立ち上げて、ブログページを開きます。

![](https://storage.googleapis.com/zenn-user-upload/ce3e19147ed3-20220604.png)

👍 ボタンを押すと ... 

![](https://storage.googleapis.com/zenn-user-upload/9d46c15c10e9-20220604.png)

いいねが通知されました！

![](https://storage.googleapis.com/zenn-user-upload/6706c8f4a86a-20220604.png)

# Vercel に deploy

https://vercel.com/new からデプロイしてみます。

![](https://storage.googleapis.com/zenn-user-upload/6397ee7b3c6d-20220604.png)

環境変数の登録を忘れずに。

![](https://storage.googleapis.com/zenn-user-upload/946e9a55aa33-20220604.png)

無事に公開されました。

https://nextjs-tutorial-blog-iine-button.vercel.app/

# おわりに

こちらサンプルコードです。

https://github.com/n1sym/nextjs-tutorial-blog-iine-button