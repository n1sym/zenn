---
title: "Nuxt+TypeScript で Vuex/axios を型安全にする"
emoji: "⛑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nuxtjs","typescript"]
published: true
---

# はじめに

https://typescript.nuxtjs.org/ja

を参考に進めていたのですが、必要最低限のことしか記述されておらず中々に苦労したので、備忘録がてら残しておきます。

セットアップは公式ドキュメント通りに行っています。また、`@vue/composition-api` プラグインを使用しています。

```ts:plugins/composition-api.ts
import Vue from 'vue'
import VueCompositionApi from '@vue/composition-api'

Vue.use(VueCompositionApi)
```

~~~js:nuxt.config.js
export default {
  plugins: ['@/plugins/composition-api']
}
~~~

# TODOリストを作る(Vuex)

こちらが完成物です。

https://nuxt-ts-sample.netlify.app/todolist/

まずはシンプルなTODOリストを作っていきます。

```ts:pages/todolist.vue
import { reactive } from "@vue/composition-api";

interface Todo {
  todo: string,
  todos: string[]
}

export default {
  setup() {
    const state = reactive<Todo>({
      todo: '',
      todos: []
    })
    const addTodo = () => {
      state.todos.push(state.todo)
      state.todo = ''
    }
    const removeTodo = (index: number) => state.todos.splice(index,1)

    return {
      state, addTodo, removeTodo
    }
  }
}
```

ストア(Vuex)を導入して状態管理を試みます。

https://typescript.nuxtjs.org/ja/cookbook/store/

公式のドキュメントを参考に進めていきます。そもそもvuexが型推論と相性が悪いようなので、色々と過程を踏む必要があるらしい。まずは`vuex-module-decorators`をインストール。

```
npm install -D vuex-module-decorators
```

モジュールとして、store の内容を記述します。

~~~ts:store/todo.ts
import { Module, VuexModule, Mutation } from 'vuex-module-decorators'

@Module({
  name: 'todo',
  stateFactory: true,
  namespaced: true
})
export default class Todos extends VuexModule {
  private todos: string[] = ['task1']

  public get getTodos () {
    return this.todos
  }

  @Mutation
  public add (todo: string) {
    this.todos.push(todo)
  }

  @Mutation
  public remove (id: number) {
    this.todos.splice(id, 1)
  }
}
~~~

アクセサーを作ります。新たに`store`を作りたいときはここに追記していく必要があります。

~~~ts:utils/store-accessor.ts
/* eslint-disable import/no-mutable-exports */
import { Store } from 'vuex'
import { getModule } from 'vuex-module-decorators'
import Todo from '~/store/todo'

let TodoStore: Todo
function initialiseStores (store: Store<any>): void {
  TodoStore = getModule(Todo, store)
}

export { initialiseStores, TodoStore }
~~~

`store/index.ts`を作っておきます。これでcomponentから TodoStore が参照できるようになりました。

~~~ts:store/index.ts
import { Store } from 'vuex'
import { initialiseStores } from '~/utils/store-accessor'
const initializer = (store: Store<any>) => initialiseStores(store)
export const plugins = [initializer]
export * from '~/utils/store-accessor'
~~~

TODOリストのコンポーネントを store 仕様に書き換えます。

```ts:pages/todolist.vue
import { defineComponent, reactive, computed } from '@vue/composition-api'
import { TodoStore } from '~/store'

interface Todo {
  todo: string
}

export default defineComponent({
  setup () {
    const state = reactive<Todo>({
      todo: ''
    })
    const todos = TodoStore
    const todolist = computed(() => todos.getTodos)

    const addTodo = () => {
      todos.add(state.todo)
      state.todo = ''
    }
    const removeTodo = (index: number) => {
      todos.remove(index)
    }

    return {
      state, todolist, addTodo, removeTodo
    }
  }
})
```

これにて完成です。

# ランダム猫画像(axios)

https://nuxt-ts-sample.netlify.app/axios

ランダムで猫の画像が表示されるページを作ってみます。

axios でも型を使いたいので、Vuexの時と同様に色々と設定していきます。

```
npm install @nuxtjs/axios
```

~~~ts:plugins/axios-accessor.ts
import { Plugin } from '@nuxt/types'
import { initializeAxios } from '~/utils/api'

export const accessor: Plugin = ({ $axios }): void => {
  initializeAxios($axios)
}

export default accessor
~~~

~~~ts:utils/api.ts
/* eslint-disable import/no-mutable-exports */
import { NuxtAxiosInstance } from '@nuxtjs/axios'

let $axios: NuxtAxiosInstance

export function initializeAxios (axiosInstance: NuxtAxiosInstance): void {
  $axios = axiosInstance
}

export { $axios }
~~~

~~~js:nuxt.config.js
  plugins: [
    '@/plugins/composition-api',
    '@/plugins/axios-accessor'
  ],
~~~

これでどこからでも axios が参照できるようになりました。

`https://aws.random.cat/meow`

このAPIを呼び出して、猫画像を取得してみます。（CSSライブラリとしてTailwindCSSを使用しています）

~~~ts:pages/axios.vue
<template>
  <div class="justify-center items-center text-center mx-auto">
    <h1>randomCat (axios)</h1>
    <button class="bg-green-500 hover:bg-green-700 text-white font-bold py-2 px-4 rounded" @click="changeCat">
      change 🐱
    </button>
    <br><br>
    <img :src="state.cat_url" alt="">
  </div>
</template>

<script lang="ts">
import { onMounted, defineComponent, reactive } from '@vue/composition-api'
import { $axios } from '~/utils/api'

export default defineComponent({
  setup () {
    const state = reactive({
      cat_url: '' as string
    })
    const asyncFunc = async (): Promise<void> => {
      const CatObj: {file: string} = await $axios.$get('https://aws.random.cat/meow')
      state.cat_url = CatObj.file
    }
    const changeCat = () => { asyncFunc() }

    onMounted(() => { asyncFunc() })

    return {
      state, changeCat
    }
  }
})
</script>
~~~

これにて完成です。

# おわりに

こちらサンプルコードです。

https://github.com/hukurouo/nuxt-ts-sample