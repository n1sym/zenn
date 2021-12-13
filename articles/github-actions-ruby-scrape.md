---
title: "GitHub ActionsでRubyによるスクレイピングを定期実行する"
emoji: "⏲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ruby","githubactions"]
published: true
---

:::message
この記事は [GitHub Actions Advent Calendar 2021](https://qiita.com/advent-calendar/2021/github-actions) 14日目の記事です。
:::

# はじめに

- 任意のページをスクレイピングしてデータを取得する
    - 言語はRubyを使用
    - JavaScriptで描画されているページは`'selenium-webdriver'`を、それ以外のページは`mechanize`を使用してスクレイピングを行う
- 取得したデータをjsonファイルで出力
- 上記のjsonファイルをGitHubリポジトリに追加する

これらを GitHub Actions で自動化してみようという記事です。

# ローカル環境でスクレイピングを実行

まずはRubyの実行環境をDockerで用意します。

Rubyのバージョンは[公式のサンプルコード](https://docs.github.com/ja/actions/automating-builds-and-tests/building-and-testing-ruby)で2.6を使っていたのでそれに倣った形です。今調べてみたら3系もちゃんと使えるみたいですね。

~~~Dockerfile:Dockerfile
FROM ruby:2.6-alpine
WORKDIR /myapp
COPY . /myapp

RUN apk add --no-cache alpine-sdk build-base bash\
  && gem install bundler:2.0.2 \
  && bundle install
  
CMD /bin/sh -c "while sleep 1000; do :; done"
~~~

最後のコマンドは、rubyコンテナを常時起動させるための呪文です。

Gemfileも書いて、
~~~rb:Gemfile
source "https://rubygems.org"

gem 'mechanize'
gem 'selenium-webdriver'
~~~

~~~
touch Gemfile.lock
~~~

selenium のコンテナも用意しておきます。

~~~yml:docker-compose.yml
version: "3.9"
services:
  chrome:
    image: selenium/standalone-chrome:3.141.59-vanadium
    hostname: selenium-server
    ports:
      - 4444:4444
    volumes:
      - /dev/shm:/dev/shm
  ruby:
    build: .
    environment:
      SELENIUM_HOST: "selenium-server:4444"
    volumes:
      - .:/myapp
~~~

コンテナを立ちあげて、準備完了。

~~~
docker-compose build
docker-compose up -d
~~~

## selenium-webdriver

まずは `'selenium-webdriver'` を使用したスクレイピングの試運転をします。今回は、中央競馬で開催されるレースについての情報を [netkeiba](https://race.netkeiba.com/top/race_list.html) から取得してみます。

参考：https://www.selenium.dev/documentation/webdriver/

~~~rb:sample.rb
require 'selenium-webdriver'

def selenium_options
    options = Selenium::WebDriver::Chrome::Options.new
    options.add_argument("--no-sandbox")
    options.add_argument("--headless")
    options.add_argument("--disable-dev-shm-usage")
    options
end
  
def selenium_capabilities_chrome
    Selenium::WebDriver::Remote::Capabilities.chrome
end

def main()
    puts "start scrape"
    caps = [
      selenium_options,
      selenium_capabilities_chrome
    ]
    driver = Selenium::WebDriver.for(:remote, capabilities: caps, url: "http://#{ENV.fetch("SELENIUM_HOST")}/wd/hub")
    driver.manage.timeouts.implicit_wait = 30
    sleep 1 # 連続アクセスを行わないように、URLを開く前には必ずsleepを入れておく。
    driver.navigate.to "https://race.netkeiba.com/top/race_list.html"
    puts driver.title

    elements = driver.find_elements(:xpath, '//dl[@class="RaceList_DataList"]/dd/ul/li/a')
    race_id_list = []
    elements.each do |e|
      race_id_list << e.attribute("href").gsub(/[^\d]/, "").to_i
    end
    puts "finished scrape race_id_list"
    puts race_id_list.uniq
end

main()
~~~

スクリプトを起動。

~~~
docker-compose run --rm ruby ruby sample.rb
~~~

無事にデータが取得できています。

~~~
Creating backend_ruby_run ... done
start scrape
レース一覧 | レース情報(JRA) - netkeiba.com
finished scrape race_id_list
202106050401
202106050402
202106050403
...
~~~

## mechanize

`selenium-webdriver`を使ったスクレイピングは、JavaScriptで描画されたページからもデータを取得できるのが強みですが、やはりブラウザをシミュレートしている分速度は遅いです。

なので、JavaScriptを使用していないページについては`mechanize`でスクレイピングを行います。レース情報のページから出走予定の競走馬のページのURLを取得しています。

~~~rb:sample2.rb
require 'mechanize'

def main()
    race_id = 202106050401
    agent = Mechanize.new
    sleep 1 # 連続アクセスを行わないように、URLを開く前には必ずsleepを入れておく。
    page = agent.get("https://race.netkeiba.com/race/shutuba.html?race_id=#{race_id}&rf=race_list")
    elements = page.xpath('//span[@class="HorseName"]/a')
    horse_url_list = []
    elements.each do |e|
      horse_url = e.attribute("href").value
      horse_url_list << horse_url
    end
    puts horse_url_list
    puts "finished scrape horse_url_list"
end

main()
~~~

スクリプトを起動。

~~~
docker-compose run --rm ruby ruby sample2.rb
~~~

こちらもデータが取得できていることを確認できました。

~~~
Creating backend_ruby_run ... done
https://db.netkeiba.com/horse/2019101448
https://db.netkeiba.com/horse/2019103610
https://db.netkeiba.com/horse/2019102480
...
~~~

# json形式でファイルを出力

GitHubにアップロードするため、データをjsonに変換して出力します。

~~~rb:output_json.rb
require "json"

arr = [
    {name: "hoge", age: 20},
    {name: "huga", age: 30}
]
file_name = "test"
File.open("#{file_name}.json","w") {|file| 
  file.puts(JSON.generate(arr))
}
~~~

~~~json:test.json
[{"name":"hoge","age":20},{"name":"huga","age":30}]
~~~

これでスクレイピングの準備が整いました。

# GitHub Actions で自動化

コード全文はこのような感じです。

~~~yml:scrape.yml
name: Scrape 1
on: 
  workflow_dispatch:
  schedule:
    - cron: '0 12 * * 5' # 金曜日の21時に起動
jobs:
  backend:
    name: ruby
    runs-on: ubuntu-latest
    services:
      selenium:
        image: selenium/standalone-chrome:3.141.59-vanadium
        ports:
          - 4444:4444
        volumes:
          - /dev/shm:/dev/shm
    env:
      SELENIUM_HOST: "localhost:4444"
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-ruby@v1
        with:
          ruby-version: '2.6' 
      - name: bundle install
        run: |
          gem install bundler:2.1.4
          bundle install
        working-directory: ${{ github.workspace }}/backend
      - name: run ruby
        run: ruby scrape.rb
        working-directory: ${{ github.workspace }}/backend
      - name: copy file
        run: cp result.json ../frontend/utils
        working-directory: ${{ github.workspace }}/backend
      - name: git setting
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
      - name: Commit files
        run: |
          git add -A
          if ! git diff-index --quiet HEAD --; then git commit -a -m "Update json (By GitHub Actions)"; fi;
          git push
      - run: echo "🎉 The job was automatically triggered by a ${{ github.event_name }} event."
~~~

詳しく見ていきます。

~~~yml
name: Scrape 1
on: 
  workflow_dispatch:
  schedule:
    - cron: '0 12 * * 5' # 金曜日の21時に起動
~~~

`name:` はアクションの名前を入力するところです。リポジトリ内で一意じゃないと怒られるので気を付けましょう。

`on:` ではアクションの起動タイミングを設定します。[ドキュメント](https://docs.github.com/ja/actions/learn-github-actions/events-that-trigger-workflows)を見るといろんなイベントが登録できるようですね。今回はUIから手動実行が可能になる `workflow_dispatch:` と、定期実行を設定するため `schedule:` の`cron:` を設定しています。cronはUTC時間で設定する必要があるのでちょっとややこしいです。

~~~yml
jobs:
  backend:
    name: ruby
    runs-on: ubuntu-latest
    services:
      selenium:
        image: selenium/standalone-chrome:3.141.59-vanadium
        ports:
          - 4444:4444
        volumes:
          - /dev/shm:/dev/shm
    env:
      SELENIUM_HOST: "localhost:4444"
~~~

`runs-on:` ではジョブを動かすための仮想環境を指定できます。基本的には、`ubuntu-latest`でよさそうです。`services:`と`env:`でseleniumのコンテナの準備をしておきます。

~~~yml
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-ruby@v1
        with:
          ruby-version: '2.6' 
      - name: bundle install
        run: |
          gem install bundler:2.1.4
          bundle install
        working-directory: ${{ github.workspace }}/backend
      - name: run ruby
        run: ruby scrape.rb
        working-directory: ${{ github.workspace }}/backend
      - name: copy file
        run: cp result.json ../frontend/utils
        working-directory: ${{ github.workspace }}/backend
~~~

rubyをインストール、gemをインストール、スクリプトを起動、出力したファイルを任意の場所へコピーという流れになります。スクレイピング類のコードを`/backend`、ページ作成関連を`/frontend`に置いているので、`working-directory:`でコマンドを起動する場所を指定しています。

~~~yml
      - name: git setting
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
      - name: Commit files
        run: |
          git add -A
          if ! git diff-index --quiet HEAD --; then git commit -a -m "Update json (By GitHub Actions)"; fi;
          git push
      - run: echo "🎉 The job was automatically triggered by a ${{ github.event_name }} event."
~~~

最後にGitHubのリポジトリを更新して完了です。

# おわりに

ここに書いたことを使って作ったページはこちらでです。情報が自動更新されるようになり、便利になりました。

https://king-halo.hukurouo.com/

リポジトリはこちら。

https://github.com/hukurouo/KingHalo