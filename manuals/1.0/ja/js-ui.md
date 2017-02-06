---
layout: docs-ja
title: Javascrript UI
category: Manual
permalink: /manuals/1.0/ja/js-ui.html
---

*Work In Progress*
*以前のReact JSページは[ReactJs](reactjs.html)へ*

# Javascript UI

ビューのレンダリングをTwigなどのPHPのテンプレートエンジン等が行う代わりにサーバーサイドまたはサーバーサイドのJavascriptが行います。

以下の順で説明します。

1. JS UIアプリケーションの作成
2. リソースの作成
3. リソースの値とJS UIをバインドするための`@Ssr`アノテート


## 前提条件

 * PHP 7.1
 * Node.js
 * yarn
 * V8Js (開発ではオプション)

Note: V8JsがインストールされていないとNode.jsでJSが実行されます。

## 用語

 * **CSR** クライアントサイドレンダリング (Webブラウザで描画)
 * **SSR** サーバーサイドレンダリング (サーバーサイドのV8またはNode.jsが描画)

# JavaScript

## インストール

BEAR.Sundayプロジェクトに`koriym/ssr-module`をインストールします。

```bash
cd /path/to/MyVedor.MyApp
composer require bear/ssr-module 1.x-dev
```

Reduxのスケルトンアプリ`koriym/js-ui-skeleto`をインストールします。

```bash
composer require koriym/js-ui-skeleton 1.x-dev
cp -r vendor/koriym/js-ui-skeleton/ui .
cp -r vendor/koriym/js-ui-skeleton/package.json .
yarn install
```

## UIアプリケーションの実行

まずはデモアプリケーションを動かして見ましょう。
現れたWebページからレンダリング方法を選択してJSアプリケーションを実行します。

```
yarn run ui
```
このアプリケーションの入力は`ui/dev/config/`の設定ファイルで設定します。

```php?
<?php
$app = 'index';                   // =index.bundle.js
$state = [                        // アプリケーションステート
    'hello' =>['name' => 'World']
];
$metas = [                        // SSRでのみ必要な値
    'title' =>'page-title'
];

return [$app, $state, $metas];
```

設定ファイルをコピーして、入力値を変えてみましょう。

```
cp ui/dev/config/index.php ui/dev/config/myapp.php
```

ブラウザをリロードして新しい設定を試します。

Note: Javascriptや本体のPHPアプリケーションを変更しないでUIのデータを変更できます。


## UIアプリケーションの作成


名前を受け取って挨拶を返す関数を`render`という名前（固定）の関数で作成します。

```
const render = state => (
  `Hello ${state.name}`
)
```

`ui/src/page/index/hello/server.js`として保存して、webpackのエントリーポイントを`ui/entry.js`に登録します。

```javascript?start_inline
module.exports = {
  hello: 'src/page/hello/server',
};
```

この`hello`アプリケーションを実行するためのファイルを`ui/dev/config/myapp.php`に作成します。

```php?
<?php
$app = 'hello';
$state = [
    ['name' => 'World']
];
$metas = [];

return [$app, $state, $metas];
```

以上です！ヴラウザをリロードして試してください。

今回は単純なテンプレート文字列を使っただけの合成でしたが、`render`関数の中の処理を`Redux React`や`Vue.js`などのUIフレームワークを使ってよりリッチなUIを作成できます。

```
const render = (state, metas) => (
  __SUPER_AWESOME_UI__ // SSR対応のライブラリで文字列を返す
)
```

引数の名前は変更することができますが、`render`という名前のglobel関数名は**変更ができません**。
ここまでPHPアプリケーションにノータッチです。SSRのアプリケーション開発はPHP開発と独立して行うことができます。

# PHP

## モジュールインストール

AppModuleに`SsrModule`モジュールをインストールします。

```php
<?php
use BEAR\SsrModule\SsrModule;

class AppModule extends AbstractModule
{
    protected function configure()
    {
        // ...
        $build = dirname(__DIR__, 2) . '/var/www/build';
        $this->install(new SsrModule($build));
    }
}
```

`$build`フォルダにwebpackでバンドルされたjsやcssファイルが出力されます。

## @Ssrアノテーション

リソースをSSRするメソッドに`@Ssr`とアノテートします。`app`にJSアプリケーション名が必要です。

```php?start_inline
<?php

namespace MyVendor\MyRedux\Resource\Page;

use BEAR\Resource\ResourceObject;
use BEAR\SsrModule\Annotation\Ssr;

class Index extends ResourceObject
{
    /**
     * @Ssr(app="index_ssr")
     */
    public function onGet($name = 'BEAR.Sunday')
    {
        $this->body = [
            'hello' => ['name' => $name]
        ];

        return $this;
    }
}
```

`$this->body`が`render`関数に1つ目の引数として渡されます。
CSRとSSRの値を区別して渡したい場合は`state`と`metas`でbodyのキーを指定します。

```php?start_inline
/**
 * @Ssr(
 *   app="index_ssr",
 *   state={"name", "age"},
 *   metas={"title"}
 * )
 */
public function onGet()
{
    $this->body = [
        'name' => 'World',
        'age' => 4.6E8;
        'title' => 'Age of the World'
    ];

    return $this;
}
```

実際`state`と`metas`をどのようにして渡してSSRを実現するかは`ui/src/page/index/server`のサンプルアプリケーションをご覧ください。

# PHPアプリケーションの実行設定

`ui/ui.config.js`を編集して、`public`にweb公開ディレクトリを`build`にwebpackのbuild先を指定します。
`build`はSsrModuleのインストールで指定したディレクトリと同じです。

```javascript
const path = require('path');

module.exports = {
  public: path.join(__dirname, '../var/www'),
  build: path.join(__dirname, '../var/www/build')
};
```

# PHPアプリケーションの実行

```
yarn run dev
```

ライブアップデートで実行します。
PHPファイルの変更があれば自動でリロードされ、Reactのコンポーネントに変更があればリロードなしでコンポーネントをアップデートします。ライブアップデートなしで実行する場合には`yarn run start`を実行します。

`lint`や`test`などの他のコマンドは[コマンド](https://github.com/koriym/Koriym.JsUiSkeleton/blob/1.x/README.ja.md#コマンド)をご覧ください。

## デバック

 * Chromeプラグイン [React developer tools](https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi)、[Redux devTools]( https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd)が利用できます。
 * 500エラーが帰ってくる場合は`var/log`や`curl` でアクセスしてレスポンス詳細を見てみましょう

## リファレンス

 * [Airbnb JavaScript スタイルガイド](http://mitsuruog.github.io/javascript-style-guide/)
 * [React](https://facebook.github.io/react/)
 * [Redux](http://redux.js.org/)
 * [Redux github](https://github.com/reactjs/redux)
 * [Redux devtools](https://github.com/gaearon/redux-devtools)
 * [Karma test runner](http://karma-runner.github.io/1.0/index.html)
 * [Mocha test framework](https://mochajs.org/)
 * [Chai assertion library](http://chaijs.com/)
 * [Yarn package manager](https://yarnpkg.com/)
 * [Webapack module bunduler](https://webpack.github.io/)