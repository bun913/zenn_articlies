---
title: "rails/webpackerのキャッシュバスティングの設定を確認してみました"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","cloudfront","rails","webpacker"]
published: true
---

最近はTerraformによるAWS環境構築などの何方かと言えばインフラ寄りの業務をさせてもらうことが多く、先日もCloudFront + S3のような設定を行なっていました。

よくある使い方の一つになるかと思うのですが、Railsのasset類をS3に配置し、CloudFrontにキャッシュして利用するように設定していたところ、Railsが利用しているWebpacker(webpack)においてどのようにjsやcssファイルが書き出されるのか分からない部分があったため本記事にキャッシュバスティングについて調べた内容を書き出しておきます。

## 事の発端

上記のとおり Railsのasset類(js,cssファイルなど)についてwebpakerを利用してS3に出力し、それをCloudFrontにキャッシュして利用する構成にしております。

ここで気になったのが、「jsやcssなどのファイル名が同一で、コンテンツ内容が変わった場合でもキャッシュって有効に機能するっけ？」というとても基本的な懸念でした。

Webpackで出力されるファイルが `example84624821743dafs4234rqdsa.js` みたいにハッシュ値みたいなものが付与されていることは知っていたのですが、根本の部分を理解していないと思ったため、ちゃんと調べてみることにしました。

## 結論

先に結論から書くと利用していた `rails/webpacker` の node_moduleでは問題なさそうでした！

- Webpackの configファイルの `output` セクションにて 出力するファイル名にハッシュ値をつけることができる
- rails/webpacker のnode_moduleでは `output` セクションにおいて `contenthash` を指定しているため、コンテンツ内容が変われば付与されるハッシュ値が代わり新しいファイル名となる
- そのためCloudFrontでもキャッシュが有効に作用し、問題なさそう


## 調べた内容

webpackerもwebpackの仕組みを利用していることは知っていたので、まず`webpack`から調べてみようと思いました。

`webpack filename hash` とかで検索すると、日本語でも結構な記事が出てきます。

例えば 以下のような入門記事のような

https://zenn.dev/unemployed/articles/webpack5-starts#%E3%83%8F%E3%83%83%E3%82%B7%E3%83%A5%E5%80%A4%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6

やっぱり webpackがバンドルしてファイルを出力する際に、ハッシュ値をつけられるようなのですが、以下3種類があるようでしたので違いを知るために公式のサイトを確認しました。（余談ですがとっかかりは2次、3次情報で良いのですが、必ず1次情報を確認するようにしています）

- hash
- contenthash
- chunkhash


公式のOutputセクションに説明が記載されていました。

https://webpack.js.org/guides/caching/#output-filenames

> We can use the output.filename substitutions setting to define the names of our output files. Webpack provides a method of templating the filenames using bracketed strings called substitutions. The [contenthash] substitution will add a unique hash based on the content of an asset. When the asset's content changes, [contenthash] will change as well.
> Let's get our project set up using the example from getting started with the plugins from output management, so we don't have to deal with maintaining our index.html file manually:

翻訳すると以下のような感じになります。

> output.filename substitutionsの設定を使って、出力ファイルの名前を定義することができます。Webpackは、substitutionと呼ばれる括弧付きの文字列を使用してファイル名をテンプレート化する方法を提供します。contenthash]置換は、アセットのコンテンツに基づいた一意のハッシュを追加します。アセットのコンテンツが変更されると、[contenthash] も同様に変更されます。
> 出力管理からプラグインを始めるの例を使ってプロジェクトをセットアップしてみましょう。

まだちょっとわかりにくいですね・・・

ということで `webpack hash vs contenthash` のようなワードで検索すると以下のようなサイトが出てきます。

https://medium.com/@sahilkkrazy/hash-vs-chunkhash-vs-contenthash-e94d38a32208

そこで contenthashのUsageについてみてみると

> In case of CSS, if you use name.[chunkhash].css in ExtractTextplugin, you will get same resulted hash for both css and js chunk. Now if you will change any CSS, your resulting chunkhash will not get changed. So in order to work that properly, you need to use name.[contenthash].css. So that when there is change in CSS, your path for css will change in index.html.

と記載されており、CSSも含むような内容なら `contenthash` の方が良さそうであるらしいです。
(contenthashはコンパイルに時間を要するため、本番以外の環境ではあまり利用しなさそうです)

### rails/webpackerの場合

今回の場合 CSSファイルなどもコンテンツに含めるため `contenthash` を利用していれば大丈夫そうだということがわかりましたので、早速利用していた`rails/webpacker` のソースコードを確認してみました。

https://github.com/rails/webpacker/blob/master/package/environments/base.js

↑のbase.js をベースに各環境(develop,staging,production)と設定を使い分けることができるようです。

今回のプロダクトでは `rails/webpacker` の設定をデフォルトで使用しているようでしたので、 `base.js` の outputセクションを確認してみると

```js
// Don't use contentHash except for production for performance
// https://webpack.js.org/guides/build-performance/#avoid-production-specific-tooling
const hash = isProduction ? '-[contenthash]' : ''
module.exports = {
  mode: 'production',
  output: {
    // 本番環境なら contenthashが利用されるようになっている
    filename: `js/[name]${hash}.js`,
    chunkFilename: `js/[name]${hash}.chunk.js`,

    // https://webpack.js.org/configuration/output/#outputhotupdatechunkfilename
    hotUpdateChunkFilename: 'js/[id].[fullhash].hot-update.js',
    path: config.outputPath,
    publicPath: config.publicPath
  },
  entry: getEntryObject(),
  resolve: {
    extensions: ['.js', '.jsx', '.mjs', '.ts', '.tsx', '.coffee'],
    modules: getModulePaths(),
    plugins: [PnpWebpackPlugin]
  },

  plugins: getPlugins(),

  resolveLoader: {
    modules: ['node_modules'],
    plugins: [PnpWebpackPlugin.moduleLoader(module)]
  },

  optimization: {
    splitChunks: { chunks: 'all' },

    runtimeChunk: 'single'
  },

  module: {
    strictExportPresence: true,
    rules
  }
}
```

本番環境であれば `contenthash` が利用され、そのほかの環境ではパフォーマンスのため `hash` が利用されていることがわかりました。

ということでコンテンツの内容が変われば、同名のファイルでもファイル名に付与されるキャッシュ値が異なるため、問題なくCloudFrontのキャッシュが効きそうであることが判明しました。

ちなみにこのような仕組みを `CacheBusting` というのですね・・・

恥ずかしながら初めて知りました・・・

https://bamch0h.hatenablog.com/entry/2019/02/18/235910

