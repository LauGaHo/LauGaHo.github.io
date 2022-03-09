# Webpack 入门

## 核心概念

### Entry

入口 ( Entry ) 指示 Webpack 以哪个文件为入口文件起点开始打包，分析构建内部依赖图。

### Output

输出 ( Output ) 指示 Webpack 打包后的资源 bundles 输出到哪里去，以及如何命名。

### Loader

加载器 ( Loader ) 让 Webpack 能够处理一些非 JavaScript 文件 ( Webpack 自身只能理解 JavaScript )

### Plugins

插件 ( Plugins ) 可以用于执行范围更广的任务。插件的范围包括：从打包优化到压缩，一直到重新定义环境中的变量等。

### Mode

模式 ( Mode ) 分为了生产模式还有开发模式，也就是 development 还有 production。



## 配置文件

1. 创建配置文件 `webpack.config.js`

2. 配置内容如下：

   ```javascript
   const { resolve } = require('path')
   
   module.exports = {
     //入口文件
     entry: './src/js/index.js',		
     // 输出配置
     output: {
       // 输出文件名
       filename: 'built.js'
       // 输出文件路径配置
       path: resolve(__dirname, 'build/js')
     },
     // 环境为开发环境
     mode: 'development'
   }
   ```

3. 运行指令：`webpack`



## 打包样式文件

1. 创建一个 CSS 文件

2. 下载并安装 npm 包：`yarn add css-loader style-loader less-loader less -dev`

3. 修改配置文件

   ```javascript
   const { resolve } = require('path')
   
   module.exports = {
     // webpack 配置
     // 入口起点
     entry: './src/index.js',
     // 输出
     output: {
       // 输出文件名
       filename: 'built.js',
       // 输出路径
       // __dirname node.js 的变量，代表当前文件的目录绝对路径
       path: resolve(__dirname, 'build')
     },
     // loader 配置
     module: {
       rules: [
         // 详细的 loader 配置
         // 不同文件必须配置不同的 loader 处理
         { 
           // 匹配哪些文件
           test: /\.css$/,
           // 使用哪些 loader 进行处理
           use: [
             // use 数组中 loader 执行顺序：从左到右，从上到下，依次执行
             'style-loader',
             // 将 css 文件变成 common.js 模块加载到 js 中，内容是样式字符串
             'css-loader'
           ]
         },
         {
           test: /\.less$/,
           use: [
             'style-loader',
             'css-loader',
             // 将 less 文件编译成 css 文件
             // 需要下载 less-loader 和 less
             'less-loader'
           ]
         }
       ]
     },
     // plugins 的配置
     plugins: [
       // 详细 plugins 的配置
     ],
     // 模式
     mode: 'development',
     // mode: 'production'
   }

	4. 运行指令：`webpack`



## 打包 HTML 资源

1. 创建文件 `index.html`

2. 下载并安装 plugins 包：`yarn add html-webpack-plugins -dev`

3. 修改配置文件：

   ```javascript
   const { resolve } = require('path')
   const HtmlWebpackPlugin = require('html-webpack-plugin')
   
   module.exports = {
     entry: './src/index.js',
     output: {
       filename: 'built.js',
       path: resolve(__dirname, 'build')
     },
     module: {
       rules: [
         // loader 配置
       ]
     },
     plugins: [
       // plugins 配置
       // html-webpack-plugin
       // 功能：默认创建一个空的 HTML，自动引入打包输出的所有资源 (JS/CSS)
       // 需求：需要有结构的 HTML 文件
       new HtmlWebpackPlugin({
         // 复制 './src/index.html' 文件，并自动引入打包输出的所有资源 (JS/CSS)
         template: './src/index.html'
       })
     ],
     mode: 'development',
   }
   ```

4. 运行指令：`webpack`



## 打包图片资源

1. 创建文件，例子中使用如下图片： `angular.jpg`、`react.png`、`vue.jpg` 

2. 下载安装对应依赖：`yarn add html-loader url-loader file-loader -dev`

3. 修改配置文件如下：

   ```javascript
   const { resolve } = require('path')
   const HtmlWebpackPlugin = require('html-webpack-plugin')
   
   module.exports = {
     entry: './src/index.js',
     output: {
       filename: 'built.js',
       path: resolve(__dirname, 'build')
     },
     module: {
       rules: [
         {
           test: /\.less$/,
           // 需要使用多个 loader 处理 less
           use: ['style-loader', 'css-loader', 'less-loader']
         },
         {
           // 默认处理不了 html 中的 img 图片
           // 处理图片资源
           test: /\.(jpg|png|gif)$/,
           // 使用一个 loader
           // 下载安装 url-loader 和 file-loader
           // 这里需要注意的是：因为 url-loader 依赖 file-loader，所以需要下载 file-loader
           loader: 'url-loader',
           options: {
             // 图片体积小于 8KB，就会被 base64 处理
             // 优点：减少请求数量，减轻服务器压力
             // 缺点：使用了 base64 之后，图片的体积会比原来大
             limit: 8 * 1024,
             // 问题：因为 url-loader 默认使用 ES6 模块化解析，而 html-loader 引入图片则是通过 common.js
             // 导致解析时会出现问题：[object Module]
             // 解决：关闭 url-loader 中的 ES6 模块化，使用 common.js 解析
             esModule: false,
             // 给图片进行重命名
             // [hash:10] 取图片的 hash 值的前 10 位
             // [ext] 取文件原来的扩展名
             name: '[hash:10].[ext]'
           }
         },
         {
           test: /\.html$/,
           // 处理 html 文件的 img 图片 (负责引入 img，从而能被 url-loader 进行处理)
           loader: 'html-loader'
         }
       ]
     },
     plugins: [
       new HtmlWebpackPlugin({
         template: './src/index.html'
       })
     ],
     mode: 'development'
   }
   ```

4. 运行指令：`webpack`



## 打包其他资源

1. 创建各种格式的文件，如：`eot`、`svg`、`ttf`、`woff`

2. 修改配置文件：

   ```javascript
   const { resolve } = require('path')
   const HtmlWebpackPlugin = require('html-webpack-plugin')
   
   module.exports = {
     entry: './src/index.js',
     output: {
       filename: 'built.js',
       path: resolve(__dirname, 'build')
     },
     module: {
       rules: [
         {
           test: /\.css$/,
           use: ['style-loader', 'css-loader']
         },
         // 打包其他的资源 (除了 html/js/css 资源以外的资源)
         {
           // 排除 css/js/html 资源
           exclude: /\.(css|html|js|less)$/,
           loader: 'file-loader',
           options: {
             name: '[hash:10].[ext]'
           }
         }
       ]
     },
     plugins: [
       new HtmlWebpackPlugin({
         template: './src/index.html'
       })
     ],
     mode: 'development'
   }
   ```

3. 运行指令：`webpack`



## devServer

1. 修改配置文件

   ```javascript
   const { resolve } = require('path')
   const HtmlWebpackPlugin = require('html-webpack-plugin')
   
   module.exports = {
     entry: './index.js',
     output: {
       filename: 'built.js',
       path: resolve(__dirname, 'build')
     },
     module: {
       rules: [
         {
           test: /\.css$/,
           use: ['style-loader', 'css-loader']
         },
         // 打包其他资源，除了 js/css/html/less 文件之外
         {
           // 排除 css/js/html/less 资源
           exclude: /\.(css|less|html|js)$/,
           loader: 'file-loader',
           options: {
             name: '[hash:10].[ext]'
           }
         }
       ]
     },
     plugins: [
       new HtmlWebpackPlugin({
         template: './src/index.html'
       })
     ],
     mode: 'development',
     devServer: {
       // 项目构建后路径
       contentBase: resolve(__dirname, 'build'),
       // 启动 gzip 压缩
       compress: true,
       // 端口号
       port: 3000,
       // 自动打开浏览器
       open: true
     }
   }
   ```

2. 运行指令：`npx webpack-dev-server`



## 提取 CSS 成单独文件

1. 下载并安装对应依赖：`yarn add mini-css-extract-plugin -dev`

2. 修改配置文件：

   ```javascript
   const { resolve } = require('path')
   const {}
   ```

   