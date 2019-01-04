---
title: "XBMC插件之网易公开课"
date: 2013-07-23
lastmod: 2013-07-23
draft: true
tags: ["tech", "python", "xmbc", "raspberrypi"]
categories: ["tech"]
description: "自己为XBMC写的一个网易公开课插件，满足课程分类/学校分类/查找功能"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
# 配置
## vue-webpack
### config/index.js [^1]
``` javascript
module.exports = {
  build: {
    index: path.resolve(__dirname, 'dist/index.html'),
    assetsRoot: path.resolve(__dirname, 'dist'),
    assetsSubDirectory: 'static',
    assetsPublicPath: '/',
    productionSourceMap: true
  },
```
assetsRoot表示编译输出目录, assetsSubDirectory定义资源文件输出的前置路径, assetsPublicPath表示url前缀, 例如"http://localhost/prefix/static/js/a.js"中的prefix.

### build/webpack.prod.conf.js

``` javascript
filename: utils.assetsPath('js/[name].[chunkhash].js'),
chunkFilename: utils.assetsPath('js/[id].[chunkhash].js')
```
这个路径结合前面的assetsRoot和assetsSubDirectory, 定义了文件的编译输出路径.

具体参考build/utils.js:

``` javascript
exports.assetsPath = function (_path) {
  var assetsSubDirectory = process.env.NODE_ENV === 'production'
    ? config.build.assetsSubDirectory
    : config.dev.assetsSubDirectory
  return path.posix.join(assetsSubDirectory, _path)
}
```

[^1]: [vue-webpack](http://hq5544.github.io/vue-webpack/backend.html)
