# 1. 背景
最近、Spring Frameworkを使用したプロジェクトをやっています。
サーバーサイド側のJavaはEclipseでの開発をおこなっておりますが、
フロントエンドにJavascriptのフレームワークであるVue.jsを採用することとなったため
フロントエンド側ではnpm + webpackを使用した開発環境になります。

2つの環境を使い分けるのは不便であるため、EclipseでもJavascriptの編集及びトランスパイルを行う方法を探しました。

# 2. 本記事でできること
Eclipse上でJavaとJavascriptの編集を行い、Javascriptのトランスパイル結果がEclipseに反映されますので
ビルド後はブラウザの更新によりJavascriptの修正内容が反映されます。

方法として、
1. webpack --watchの結果をEclipseに反映させる
2. Eclipseのビルダーを作成してwebpackによるトランスパイルを行う
の2つの方法を紹介します。

以降の作業は、下記の環境ができているものとして記事を進めます。
- Eclipse Neon2  
Buildship:Gradle用プラグイン、JDT,
- node.js + vue-cli

# 3. サンプルの準備

## 3.1. サーバーサイド側のSpringソースの準備

サーバー側では、SpringBootを使用して静的ページをデプロイします。

[Spring Initializr](http://start.spring.io/) でボイラーテンプレートを作成します。

左側のDependenciesに「web」「DevTools」を入力して「Generate Project」をクリックします。
プロジェクトソースをダウンロードして任意のフォルダで展開します。

DevToolsをいれておくとEclipseでソースを修正保存したときにホットデプロイができて捗ります

## 3.2. フロントエンド側のVue.jsソースの準備

vue-cliを使ってボイラーテンプレートを作成します。
```
vue init webpack-simple 
```
ソースが出来上がったら、サーバーサイドで展開したプロジェクトフォルダの中にVue.jsのファイルを配置します。

- index.htmlは、src/main/resources/staticに移動
- assets App.vue main.js は src/main/javascriptを作成してその中に移動

## 3.3. フォルダ構成
上記の作業で下記のようなフォルダ階層になります。

```
.babelrc
.gitignore
.build.gradle
 gradlew
 gradlew.bat
 package.json
 README.md
 webpack.config.js
 src
  └─main
     ├─java
     │  └─com
     │      └─example
     │              VuetestApplication.java
     │
     ├─javascript
     │  │  App.vue
     │  │  main.js
     │  │
     │  └─assets
     │        logo.png
     │
     └─resources
         │  application.properties
         │
         ├─static
         │  │  index.html
         │  │
         │  └─dist <== webpackにより出力されるフォルダ
         │        build.js
         │        build.js.map
         │        logo.png
         │
         └─templates
```

## 3.3. Eclipseにプロジェクトを取り込む
Gradleプロジェクトの取り込みでEclipseに取り込みます。

---

# 4. EclipseでJavascriptを編集するための設定
Eclipseでmain.jsをエディタで表示すると 1-2行目の import文でエラーとなってしまいます。
モデュールとして認識させるため以下の設定を行います。

### 4.1. プロジェクトファセットにJavascriptを追加
プロジェクトのプロパティで「プロジェクト・ファセット」を選択し「JavaScript」にチェックを入れて「OK」をクリックする。

### 4.2. バリデーターの設定
ワークスペースの設定またはプロジェクトのプロパティで「Javascript」→「検証」を選択しソースタイプを「モジュール」に変更して「OK」をクリックする。

# 5. Webpackでトランスパイルする設定
配置したVue.jsのファイルに対してwebpack.configの entry, outputのフォルダを調整します。

```javascript:webpack.config
var path = require('path')
var webpack = require('webpack')

module.exports = {
  entry: './src/main/javascript/main.js',
  output: {
    path: path.resolve(__dirname, './src/main/resources/static/dist'),
    publicPath: '/dist/',
    filename: 'build.js'
  },
以下略 ...
```

# 6. Eclipseで外部で変更されたファイルを自動認識する
まずはEclipse外でトランスパイルをして変更されたファイルをEclipseに認識させる方法です。
利用方法としてはEclipseと別にターミナルを開いて、Webpackでファイルの変更監視をして
トランスパイルされたファイルをEclipseに認識させファイルツリーに反映させます。

## 6.1. Eclipseによるファイルの変更監視の設定
Eclipseのエディタ以外で変更されたファイルをEclipseのファイルツリーに自動で反映させるには

Eclipseの環境設定 -> 一般 -> ワークスペース にある
* ネイティブフックまたはポーリングを利用してリフレッシュ
* アクセス時にリフレッシュ

の2つにチェックを入れます。

## 6.2. npmタスクの設定
webpackによるファイル監視タスクを追加します。

scripts に watchタスクを追加します。
NODE_ENVについては、productionでもdevelopmentでもご自由にどうぞ

```json:package.json
  "scripts": {
    "dev": "cross-env NODE_ENV=development webpack-dev-server --open --inline --hot",
    "build": "cross-env NODE_ENV=production webpack --progress --hide-modules",
    "watch": "cross-env NODE_ENV=production webpack --progress --watch"
  },
以下略 ...
```
ターミナルを開いて

```text
npm run watch
```
としておくことで、webpackによるトランスパイルを行い、eclipseのファイルツリーに最新が反映されます。

---

# 7. Eclipseの自動ビルドでトランスパイルする
次に紹介するのが今回の目的のJavascriptの変更監視にWebpackを使用しないでEclipseの自動ビルドでトランスパイルをする方法です。

## 7.1. トランスパイル用のシェル、バッチファイルを作成

トランスパイルするためのシェル、またはバッチファイルを用意しプロジェクトルートに置きます。

```bat:npmbuild.bat
npm run build
```
## 7.2. npmビルダーの作成
1. プロジェクトのプロパティで「ビルダー」を選択し「新規」をクリックする。
1. 「作成する外部ツール・タイプの選択」で「プログラム」を選択する。
1. 上部の「名前」は適当な名前をつける。

### 7.2.1 メインタブ
1. 「ロケーション」は先ほど作成したトランスパイル用のシェル、バッチファイルを指定する。  
`${workspace_loc}/${project_name}/npmbuild.bat`
1. 「作業ディレクトリ」はプロジェクトルートを指定する。  
`${project_loc:}`

### 7.2.2 リフレッシュタグ
1. 「完了時にリソースをリフレッシュ」にチェックを入れる。
1. 「特定のリソース」を選択し、「リソースの指定」でトランスパイルで出力されるフォルダを指定する。また「再帰的にサブフォルダを組み込む」にチェックを入れる。

### 7.2.3 ビルドオプション
1. 「手動ビルド中」、「自動ビルド中」にチェックを入れる。
1. 「関連するリソースのワーキングセットを指定」にチェックを入れ、「リソースの指定」でjavascriptのソースフォルダを指定する。

### 参考
- [SpringBootの開発に自動リロード(ホットデプロイ)を導入する](http://endok.hatenablog.com/entry/2016/06/11/151612)
- [Vue.js を vue-cli を使ってシンプルにはじめてみる](http://qiita.com/567000/items/dde495d6a8ad1c25fa43)
- [もうgulpやwebpackで消耗しない！vue-cliを使ったVue.js開発 ](http://kuroeveryday.blogspot.jp/2016/09/vue-cli.html)
- [Eclipse NEON で ES2015 module を有効にする](http://qiita.com/hidekatsu-izuno/items/6dfe870e3fbeb7823746)
- [Eclipseの外部で変更されたファイルを自動的に認識する方法](http://qiita.com/yakumo/items/97eed416c4720c5e25d8)
