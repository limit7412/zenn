---
title: "vue-cli + axios でmock環境を雑に作ったのでメモ"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["axios","JavaScript","vue"]
published: false
---

vue-cliで開発するのにローカルのapiとmockを切り替えながら開発する環境を作ったが絶対忘れるので自分用にメモ
ローカルでの開発中に `yarn run serve` でapiにつないで `yarn run mock` でmockを見れるようにすることを目指します

## コマンド
まずコマンドの方から先に仕込んで行きます

```json:package.json
  "scripts": {
    "serve": "vue-cli-service serve",
    "mock": "vue-cli-service serve --mode mock",
    "build": "vue-cli-service build",
```
起動時に `--mode mock` をつけることで環境変数を読み込む元のファイルを `.env.development` から `.env.mock` に変えることが出来るのでこれを `package.json` に仕込んで `yarn run mock` で起動出来るようにします

今回は環境変数をそれぞれ以下の通りにします

```.env.development
VUE_APP_USE_MOCK=false
```

```.env.mock
VUE_APP_USE_MOCK=true
```
この環境変数を使ってmock環境かどうか判定します
mockを利用したい環境ではtrue、そうでないときはfalseにします

## axios
次にapiへのアクセス部分を書いていきます

```js:api.js
import axios from 'axios'
import mock from './mock/api';

export const client = axios.create({
  baseURL: <apiのurl>,
})

// モック起動
if (JSON.parse(process.env.VUE_APP_USE_MOCK)) {
  mock.run(client)
}

export default {
  getApi (params, cb, err_cb) {
    client.get('/', headers)
      .then(response => {
        cb(response.data)
      })
      .catch(error => {
        console.log(error)
        err_cb(error)
      })
  },
  postApi (params, cb, err_cb) {
    client.post('/', params, headers)
      .then(response => {
        cb(response.data)
      })
      .catch(error => {
        err_cb(error)
      })
  },
}
```

apiへのアクセスはaxiosを使っていて基本何か特別なことは無いです
ただ先程の環境変数がtrueな場合、mockを起動しています
(ここでは環境変数の判定をするのにJSON.parseを使っているのは値をbooleanに変換するためですが、正直あまりいい方法では無いと思う…）

## mock
最後にmockの振る舞いを定義します
今回はパスを `./mock/api` としています

```js:mock/api.js
import MockAdapter from 'axios-mock-adapter';

const initialData = () => ([
  { id: '1', name: '島村卯月', notes: '' },
  { id: '2', name: '渋谷凛', notes: '' },
  { id: '3', name: '本田未央', notes: '' },
])
let data = initialData()

export default {
  run : client => {
    const mock = new MockAdapter(client)

    mock.onGet('/').reply(200, data)
    mock.onPost('/').reply(
      config => {
        let res = JSON.parse(config.data)

        res.id = Number(data.length) + 1
        data.push(res)
        return [200, res]
      }
    )
  },
  getData: () => {
    return JSON.parse(JSON.stringify(data))
  },
  setData: input => {
    data = JSON.parse(JSON.stringify(input))
  },
  initialize: () => {
    data = initialData()
  },
}
```

作り込んではいないので雑ですが見ていきましょう
今回は `axios-mock-adapter` を利用します
`initialData` はmock用の初期データです
これを初期化時に実際のmockのデータを保存する変数である `data` に突っ込みます
`run` は起動する時に実行される関数で、ここにmockがどういった振る舞いをするかを定義しています
ここではgetでは現在のデータをそのまま、postでは追加してそのまま入ってきた物をそのまま返却しています
`getData` `setData` `initialize` ではテスト等でデータを見たりいじったりしています

あとはコマンドを最初に設定した通りに叩いてやればmock環境でvue-cliが起動します

```
$ yarn run mock
```
