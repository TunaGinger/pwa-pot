# 好奇心駆動PoT！ 投稿内のURL先を取得するやつ
## 使いたいブラウザAPI&キャッシュ戦略

- Service Worker（以下swなど）
- Cache API
  - キャッシュのみに問い合わせ (静的キャッシュしたApp Shell)
  - キャッシュ優先で問い合わせ (images)
- IndexedDB できれば生APIで
  - indexedDBとネットワークに同時に問い合わせ (json)
  - ローカルな投稿もindexedDBに格納
- 通知
  - ネットワーク経由で取得したデータがあったとき (images, json)
  - サーバからのPushはしない
  - 一番有用性のない部分（通知すべきイベントが発生するのがアプリを開いてるとき）

特に動的キャッシュについては、表側のJSでも扱いたいし、裏側のSWでも扱いたい。  
ということで画像とjsonで分けた。そしてjsonはindexedDBになった（なる予定）。

## フレームワーク・ライブラリ

- React + Redux
- BootstrapのCSS (CDN)

とりあえず過剰でも良いので雑にReact Hooksを使ってみたい。  
TypeScriptも使ってみたいけど多分使わない。

## いかにシンプルに実現するか

- SPAのホスティングのみ。
- APIサーバ等は立てない。
- デザインはシンプルに
  - 投稿フォームはテキストエリアひとつ + ボタン
  - 下にずらっと投稿済みを表示
- 投稿text内のurlを取得して画像か否か判別
  - 画像だったらimgタグ
  - それ以外は `Accept: 'application/json'` で GETリクエスト
- textと取得したデータが並んで表示される

## ナイーブに内容を詰めてみる
### ナイーブに想定するredux stateの最終的なカタチ

```js
{
  posts: [
    {
      id: string, // key
      text: string,
      urls: []string
    }
  ],
  urlData: {
    [url]: {       // key
      url: string, // key
      isImage: boolean|null, // 画像の場合はここまで
      fetchStatus: 'fetching'|'network'|'indexedDB'|'error',
      data: string
    }
  }
}
```

※ フォームは React で制御する（多分）

### ナイーブに想定するindexedDBの最終的なカタチ

```
- posts: []
  - {}
    - id: string
    - text: string
    - urls: []string
- jsonData: []
  - {}
    - url(key): string
    - data: string
```

- url先を画像と判定したらimgタグを利用し、ブラウザ（サービスワーカー）に任せる。
- 画像以外は json であると仮定し url をキーに indexedDB に格納。

### ナイーブに想定する処理の流れ

基本的にはコンポーネントのマウントがフェッチを発火するように。

#### 初めての起動
1. AppShellを静的キャッシュ
1. AppShellをキャッシュから取得

#### 起動
1. AppShellをキャッシュから取得
2. indexedDBからpostsを読み込み
3. => post用コンポーネントがマウントされる
4. => postの各urlをkeyとしてコンポーネントがマウントされる
5. 各URL先が画像かどうか判別

- 未判別     urlDataがないもしくは isImage: null
  - => 代替コンポーネント
- 画像である isImage: true
  - => imgタグ表示用コンポーネント
  - service worker がキャッシュ戦略を実行に移す
- 画像でない isImage: false
  - => json表示用コンポーネント
  - actionCreator等でキャッシュ戦略を実行に移す

6. 読み込まれたものから表示されていく

#### 投稿
1. url を取得しstoreに格納
1. => コンポーネントがマウントされる
1. 以下の流れは上の方参照

### ナイーブに想定する実装順序

1. このREADME（随時更新）
1. webpackでうまいことswを扱う
1. App Shellを静的キャッシュしてキャッシュオンリーに
1. React + Reduxでテキストの投稿を扱えるように
1. 投稿をindexedDBに格納・取得するように
1. 投稿からurlを取得して画像かどうか判別するように
1. 画像だったらimgタグに埋め込むように
1. 画像のキャッシュ戦略（キャッシュ優先）
1. jsonのフェッチ＆いい感じに表示
1. jsonのキャッシュ戦略
1. 通知許可ボタン
1. ネットワーク経由のデータ取得を通知するように