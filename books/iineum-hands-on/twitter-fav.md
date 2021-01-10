---
title: "Twitter API でツイートを取得する"
---


twitter API からいいねしたツイートを取得、データを整形して返却するコードを書いていきます。

引数として以下を受け、

| Name | Required | Description |
| ---- | ---- | ---- |
| screen_name | required | ツイッターのスクリーンネーム (例: @hukurouo) |
| count | required | 取得するツイート数。最大値は200 |
| max_id | optional | 指定したツイートIDよりも古いツイートを取得する |

以下を返却します。

~~~ts
images = {
  url: string[];    // 画像のURL 
  height: number[]; // 画像の高さ (並べる時に使う)
  source: string[]; // ツイートのURL
  max_id: number;   // 最後に取得したツイートのID（続きからツイートを取得するため）
};
~~~



全文はこのような感じになります。

~~~js:index.js
const twitter = require("twitter");

const client = new twitter({
  consumer_key: process.env.consumer_key,
  consumer_secret: process.env.consumer_secret,
  access_token_key: process.env.access_token_key,
  access_token_secret: process.env.access_token_secret,
});

exports.handler = async (event) => {
  const params = { screen_name: event.queryStringParameters.name, count: 180 };
  if (event.queryStringParameters.maxid) {
    params.max_id = Number(event.queryStringParameters.maxid);
  }
  const tweets = await getTweets(params);
  return {
    statusCode: 200,
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Headers": "Content-Type",
      "Access-Control-Allow-Methods": "OPTIONS,GET",
    },
    body: JSON.stringify(tweets),
  };
};

const getTweets = function (params) {
  return new Promise((resolve, reject) => {
    client.get("favorites/list", params, function (error, tweets, response) {
      if (!error) {
        resolve(extractData(tweets));
      } else {
        console.error(error);
        reject(error);
      }
    });
  });
};

const extractData = function (tweets) {
  var images = { url: [], source: [], height: [], max_id: 0 };
  tweets.forEach((tweet) => {
    if (tweet.entities.media) {
      if (tweet.entities.media[0].type == "photo") {
        if (!tweet.entities.media[0].media_url_https.includes("video_thumb")) {
          images.url.push(tweet.entities.media[0].media_url_https);
          images.source.push(tweet.entities.media[0].expanded_url);
          const w = tweet.entities.media[0].sizes.medium.w;
          const h = tweet.entities.media[0].sizes.medium.h;
          images.height.push(h / w);
        }
      }
    }
  });
  const max_id = tweets[tweets.length - 1].id - 10000;
  images.max_id = max_id;
  return images;
};
~~~

`count: 180` については、最大値200にすると時々取得に失敗するので若干減らしています。

` "Access-Control-Allow-Origin": "*",` では、リソースがすべてのドメインからアクセスできることを許可しています。

細かい twitter API の仕様はこちらで確認できます。

https://developer.twitter.com/en/docs/twitter-api/v1/tweets/post-and-engage/api-reference/get-favorites-list