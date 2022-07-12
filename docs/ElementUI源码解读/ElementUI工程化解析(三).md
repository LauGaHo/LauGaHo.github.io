# ElementUI工程化解析(三)

## 打包配置

### CommonJS 和 CommonJS2

CommonJS 规范只定义了 `exports`，而 `module.exports` 是 NodeJS 对 CommonJS 的实现，这种扩展实现被称为 CommonJS2。

```javascript
// CommonJS
exports.a = 'a';
exports.b = 'b';

// CommonJS2
module.exports = {
  a: 'a',
  b: 'b'
}
```

### `build/config.js`

该文件内容是打包配置的公用配置。

```javascript
var path = require('path');
var fs = require('fs');
// 排除 node_modules 目录中所有模块
var nodeExternals = require('webpack-node-externals');
// 项目组件清单
var Components = require('../components.json');
// 工具方法文件列表
var utilsList = fs.readdirSync(path.resolve(__dirname, '../src/utils'));
// 混入方法文件列表
var mixinsList = fs.readdirSync(path.resolve(__dirname, '../src/mixins'));
// 动画文件列表
var transitionList = fs.readdirSync(path.resolve(__dirname, '../src/transitions'));
// 外部扩展配置
var externals = {};

// 遍历组件
Object.keys(Components).forEach(function(key) {
  // 设置组件 element-ui/packages/{component} 到 element-ui/lib/{component} 的引入
  externals[`element-ui/packages/${key}`] = `element-ui/lib/${key}`;
});

// 国际化依赖
externals['element-ui/src/locale'] = 'element-ui/lib/locale';

// 遍历工具方法
utilsList.forEach(function(file) {
  file = path.basename(file, '.js');
  // 设置 element-ui/src/utils/{utils} 到 element-ui/lib/utils/{utils}
  externals[`element-ui/src/utils/${file}`] = `element-ui/lib/utils/${file}`;
});

// 遍历混入方法
mixinsList.forEach(function(file) {
  file = path.basename(file, '.js');
  // 设置 element-ui/src/mixins/{file} 到 element-ui/lib/mixins/{file}
  externals[`element-ui/src/mixins/${file}`] = `element-ui/lib/mixins/${file}`;
});

// 遍历动画方法
transitionList.forEach(function(file) {
  file = path.basename(file, '.js');
  // 设置 element-ui/src/transitions/{file} 到 element-ui/lib/transitions/{file}
  externals[`element-ui/src/transitions/${file}`] = `element-ui/lib/transitions/${file}`;
});

// 设置 vue 依赖 external 属性
externals = [Object.assign({
  vue: 'vue'
}, externals), nodeExternals()];

exports.externals = externals;

// 创建 import 或 require 的别名，确保模块引入变得简单
exports.alias = {
  main: path.resolve(__dirname, '../src'),
  packages: path.resolve(__dirname, '../packages'),
  examples: path.resolve(__dirname, '../examples'),
  'element-ui': path.resolve(__dirname, '../')
};

// vue 扩展依赖配置
exports.vue = {
  root: 'Vue',
  commonjs: 'vue',
  commonjs2: 'vue',
  amd: 'vue'
};

// 排除所有符合条件的模块
exports.jsexclude = /node_modules|utils\/popper\.js|utils\/date\.js/;
```

外部扩展 `external` 从输出的 `bundle` 中排除依赖，在运行时从外部获取这些依赖 `external dependencies`，主要解决组件依赖导致代码冗余问题。其中 `exports.externals = externals` 内容格式如下：

```javascript
[
  {
    "vue": "vue",
    "element-ui/packages/input": "element-ui/lib/input",
    "element-ui/packages/input-number": "element-ui/lib/input-number",
    "element-ui/packages/button": "element-ui/lib/button",
    ......
    "element-ui/src/locale": "element-ui/lib/locale",
    ......
    "element-ui/src/utils/date": "element-ui/lib/utils/date",
    "element-ui/src/utils/merge": "element-ui/lib/utils/merge",
    ......
    "element-ui/src/mixins/emitter": "element-ui/lib/mixins/emitter",
    "element-ui/src/mixins/focus": "element-ui/lib/mixins/focus",
    "element-ui/src/transitions/collapse-transition": "element-ui/lib/transitions/collapse-transition"
  }
]
```

### `build/webpack.common.js` ( `element-ui.common.js` [CommonJS] )

以 CommonJS2 规范打包构建类库。

- 调用命令：`webpack --config build/webpack.common.js`
- 入口文件：`src/index.js`
- 输出文件：以 CommonJS2 规范构建输出到 `lib/element-ui.common.js`

```javascript
const path = require('path');
// 构建进度条
const ProgressBarPlugin = require('progress-bar-webpack-plugin');
const VueLoaderPlugin = require('vue-loader/lib/plugin');
// webpack 公共配置
const config = require('./config');

module.exports = {
  // 模式配置选项，告知 webpack 使用相应模式的内置优化
  mode: 'production',
  // 入口
  entry: {
    // Entry Description
    // 若传入一个对象，对象属性值可以是一个字符串，字符串数组或者一个描述符 descriptor
    app: ['./src/index.js']
  },
  // 输出
  output: {
    // lib 的绝对路径
    path: path.resolve(process.cwd(), './lib'),
    // 为项目的所有资源指定一个基础路径
    publicPath: '/dist/',
    // 打包构建的文件名
    filename: 'element-ui.common.js',
    // 对于需要按需加载的 chunk 文件，使用该选项来控制输出
    chunkFilename: '[id].js',
    // libraryTarget 决定暴露哪些模块
    // libraryExport：default - 入口的默认导出将分配给 library target
    libraryExport: 'default',
    // 库的名称
    library: 'ELEMENT',
    // 配置将库暴露的方式
    // 类型默认包括：'var', 'module', 'assign', 'assign-properties', 'this', 'window'
    // 'self', 'global', 'commonjs', 'commonjs2', 'common-module', 'amd', 'amd-require'
    // 'umd', 'umd2', 'jsonp', 'system'
    libraryTarget: 'commonjs2'
  },
  // 解析
  resolve: {
    // 在引入模块时不带扩展
    // 尝试按照顺序解析这些后缀名，如果有多个文件有相同的名字，但是后缀名不一样，webpack 会解析在数组首位的后缀的文件，并跳过其余的后缀
    extensions: ['.js', '.vue', '.json'],
    // 创建 import 或 require 的别名，确保模块引入变得简单
    alias: config.alias,
    // 告知 webpack 解析模块时应搜索的目录
    modules: ['node_modules']
  },
  // 外部扩展
  externals: config.externals,
  // 控制 webpack 如何通知「资源 asset 和入口起点超过指定文件限制」
  performance: {
    // 不展示警告或错误提示
    hints: false
  },
  // 可以在统计输出里指定想看到的信息
  stats: {
    // 是否添加关于子模块的信息
    children: false
  },
  // 根据选择的 mode 来执行不同的优化
  optimization: {
    // 告知 webpack 使用 TerserPlugin 或其他在 optimization.minimizer 定义的插件压缩 bundle
    minimize: false
  },
  // 处理项目中的不同类型的模块
  module: {
    // 配置解析规则，使用 loader 对文件进行预处理
    rules: [
      {
        test: /\.(jsx?|babel|es6)$/,
        include: process.cwd(),
        exclude: config.jsexclude,
        loader: 'babel-loader'
      },
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          compilerOptions: {
            preserveWhitespace: false
          }
        }
      },
      {
        test: /\.css$/,
        loaders: ['style-loader', 'css-loader']
      },
      {
        test: /\.(svg|otf|ttf|woff2?|eot|gif|png|jpe?g)(\?\S*)?$/,
        loader: 'url-loader',
        query: {
          limit: 10000,
          name: path.posix.join('static', '[name].[hash:7].[ext]')
        }
      }
    ]
  },
  // 添加插件
  plugins: [
    new ProgressBarPlugin(),
    new VueLoaderPlugin()
  ]
};
```

### `build/webpack.component.js` ( `lib/[name].js` [CommonJS2] )

以 CommonJS2 规范对每个组件单独构建，支持按需引入。

- 调用命令：`webpack --config build/webpack.component.js`。
- 入口文件：`components.json` 中的组件目录。
- 输出文件：把 `packages` 目录下的组件，以 CommonJS2 规范单独构建输出到 `lib/components-name.js`。

```javascript
const path = require('path');
const ProgressBarPlugin = require('progress-bar-webpack-plugin');
const VueLoaderPlugin = require('vue-loader/lib/plugin');
// 组件清单
const Components = require('../components.json');
const config = require('./config');

const webpackConfig = {
  mode: 'production',
  // 多个入口起点
  // {
  // 	"pagination": "./packages/pagination/index.js",
  // 	"dialog": "./packages/dialog/index.js"
  // 	"autocomplete": "./packages/autocomplete/index.js",
  // 	"dropdown": "./packages/dropdown/index.js"
  // }
  entry: Components,
  output: {
    path: path.resolve(process.cwd(), './lib'),
    publicPath: '/dist/',
    // name 为入口对象中每个属性的键 (key) 也就是 component-name
    filename: '[name].js',
    chunkFilename: '[id].js',
    libraryTarget: 'commonjs2'
  },
  resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: config.alias,
    modules: ['node_modules']
  },
  externals: config.externals,
  performance: {
    hints: false
  },
  // 统计信息输出设置，没有输出
  stats: 'none',
  optimization: {
    minimize: false
  },
  module: {
    rules: [
      {
        test: /\.(jsx?|babel|es6)$/,
        include: process.cwd(),
        exclude: config.jsexclude,
        loader: 'babel-loader'
      },
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          compilerOptions: {
            preserveWhitespace: false
          }
        }
      },
      {
        test: /\.css$/,
        loaders: ['style-loader', 'css-loader']
      },
      {
        test: /\.(svg|otf|ttf|woff2?|eot|gif|png|jpe?g)(\?\S*)?$/,
        loader: 'url-loader',
        query: {
          limit: 10000,
          name: path.posix.join('static', '[name].[hash:7].[ext]')
        }
      }
    ]
  },
  plugins: [
    new ProgressBarPlugin(),
    new VueLoaderPlugin()
  ]
};

module.exports = webpackConfig;
```

### `build/webpack.conf.js` ( `lib/index.js` [UMD] )

以 UMD 规范打包构建类库，不仅可以在 NodeJS 环境使用，也可以在浏览器环境 Browser 使用，需要设置 `umdNamedDefine: true`。

- 调用命令：`webpack --config build/webpack.conf.js`。
- 入口文件：`src/index.js`。
- 输出文件：以 UMD 规范构建输出到 `lib/index.js`。

```javascript
const path = require('path');
// 构建进度条
const ProgressBarPlugin = require('progress-bar-webpack-plugin');
const VueLoaderPlugin = require('vue-loader/lib/plugin');
// Terser 代码压缩
const TerserPlugin = require('terser-webpack-plugin');

const config = require('./config');

module.exports = {
  mode: 'production',
  // 入口
  entry: {
    app: ['./src/index.js']
  },
  // 输出
  output: {
    path: path.resolve(process.cwd(), './lib'),
    publicPath: '/dist/',
    filename: 'index.js',
    chunkFilename: '[id].js',
    libraryTarget: 'umd',
    libraryExport: 'default',
    library: 'ELEMENT',
    // 将命名由 UMD 构建的 AMD 模块
    umdNamedDefine: true,
    // 将使用哪个全局对象来挂载 library
    globalObject: 'typeof self !== \'undefined\' ? self : this'
  },
  resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: config.alias
  },
  // 防止将某些 import 的包 package 打包到 bundle 中
  // 在运行时 runtime 再从外部获取这些扩展依赖
  externals: {
    // config.vue
    // {
    // 	root: 'Vue',
    // 	commonjs: 'vue',
    // 	commonjs2: 'vue',
    // 	amd: 'vue'
    // }
    vue: config.vue
  },
  optimization: {
    // 提供一个或多个定制过的 TerserPlugin 实例，覆盖默认
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          output: {
            comments: false
          }
        }
      })
    ]
  },
  performance: {
    hints: false
  },
  stats: {
    children: false
  },
  module: {
    rules: [
      {
        test: /\.(jsx?|babel|es6)$/,
        include: process.cwd(),
        exclude: config.jsexclude,
        loader: 'babel-loader'
      },
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          compilerOptions: {
            preserveWhitespace: false
          }
        }
      }
    ]
  },
  plugins: [
    new ProgressBarPlugin(),
    new VueLoaderPlugin()
  ]
};
```

#### `external` 配置

通过该方式引入的依赖库，不会打包到 bundle 中，以下任何一种形式在各种模块上下文使用：

- `root`：可以通过一个全局变量访问 library (例如通过 `script` 标签)。
- `commonjs`：可以将 library 作为一个 CommonJS 模块访问。
- `commonjs2`：和上边类似，但是导出的是 `module.exports.default`。
- `amd`：类似于 CommonJS，但使用的是 AMD 模块系统。

一个形如 `{ root, amd, commonjs, ... }` 的对象仅允许使用 `libraryTarget: 'umd'` 这样的配置。

```javascript
// 防止将某些 import 的包 package 打包到 bundle 中
// 在运行时 runtime 再从外部获取这些扩展依赖
externals: {
  // config.vue
  // {
  // 	root: 'Vue',
  // 	commonjs: 'vue',
  // 	commonjs2: 'vue',
  // 	amd: 'vue'
  // }
  vue: config.vue
},
```

生成 `lib/index.js` 中，依赖库 `vue` 引入声明代码如下：

```javascript
(function webpackUniversalModuleDefinition(root, factory) {
  // commonjs2
	if(typeof exports === 'object' && typeof module === 'object')
		module.exports = factory(require("vue"));
  // amd
	else if(typeof define === 'function' && define.amd)
		define("ELEMENT", ["vue"], factory);
  // commonjs
	else if(typeof exports === 'object')
		exports["ELEMENT"] = factory(require("vue"));
  // root
	else
		root["ELEMENT"] = factory(root["Vue"]);
})
```

### `build/webpack.demo.js`

提供了两套打包配置，生产模式用于项目网站的构建，开发模式用于组件展示测试的构建。使用了 CSS、JS 构建的优化插件，还配置了 `splitChunks` 抽取公共模块解决重复引入第三方库的问题。

`npm run deploy:build` 命令打包构建项目网站。

- 调用命令：`webpack --config build/webpack.demo.js`。
- 模式：`production`。
- 入口文件：`examples/entry.js`。
- 输出文件：构建内容输出至 `examples/element-ui/` 目录下。

`npm run dev` 命令打包运行项目网站，用于开发调试。

- 调用命令：`webpack-dev-server --config build/webpack.demo.js`。
- 模式：`development`。
- 入口文件：`examples/entry.js`
- 输出文件：构建内容输出至 `examples/element-ui/` 目录下。

`npm run dev:play` 命令用于组件库开发中的功能展示。

- 调用命令：`webpack-dev-server --config build/webpack.demo.js`。
- 模式：`development`。
- 入口文件：`examples/entry.js`。
- 输出文件：构建内容输出至 `examples/element-ui/` 目录下。

```javascript
const path = require('path');
const webpack = require('webpack');
// 抽离 CSS 单独打包成一个 CSS 文件。支持 CSS 和 SourceMaps 的按需加载。
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
// 将已经存在的单个文件或整个目录复制到构建目录。
const CopyWebpackPlugin = require('copy-webpack-plugin');
// HTML 文件的创建
const HtmlWebpackPlugin = require('html-webpack-plugin');
// 进度条插件
const ProgressBarPlugin = require('progress-bar-webpack-plugin');
// Vue-Loader 插件，用于编译 .vue 文件
const VueLoaderPlugin = require('vue-loader/lib/plugin');
// 压缩 CSS
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin');
// 压缩 JS
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');
const launchEditorMiddleware = require('launch-editor-middleware');

const config = require('./config');

const isProd = process.env.NODE_ENV === 'production';
const isPlay = !!process.env.PLAY_ENV;

const webpackConfig = {
  mode: process.env.NODE_ENV,
  // entry.js 项目网站
  // play.js 功能组件开发展示
  entry: isProd ? {
    docs: './examples/entry.js'
  } : (isPlay ? './examples/play.js' : './examples/entry.js'),
  output: {
    path: path.resolve(process.cwd(), './examples/element-ui/'),
    publicPath: process.env.CI_ENV || '',
    filename: '[name].[hash:7].js',
    chunkFilename: isProd ? '[name].[hash:7].js' : '[name].js'
  },
  resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: config.alias,
    modules: ['node_modules']
  },
  // webpack-dev-server 开发服务器配置
  devServer: {
    // 用于外部访问
    host: '0.0.0.0',
    // 指定端口号
    port: 8085,
    // 捆绑的文件将在此路径下
    publicPath: '/',
    // 启用 Webpack 的热模块替换更新 HMR
    hot: true,
    before: (app) => {
      /*
       * 编辑器类型 :此处的指令表示的时各个各个编辑器在cmd或terminal中的命令
       * webstorm
       * code // vscode
       * idea
      */
      app.use('/__open-in-editor', launchEditorMiddleware('code'));
    }
  },
  performance: {
    hints: false
  },
  stats: {
    children: false
  },
  module: {
    rules: [
      {
        enforce: 'pre',
        test: /\.(vue|jsx?)$/,
        exclude: /node_modules/,
        loader: 'eslint-loader'
      },
      {
        test: /\.(jsx?|babel|es6)$/,
        include: process.cwd(),
        exclude: config.jsexclude,
        loader: 'babel-loader'
      },
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          compilerOptions: {
            preserveWhitespace: false
          }
        }
      },
      {
        test: /\.(scss|css)$/,
        // 生产环境使用 MiniCssExtract
        use: [
          isProd ? MiniCssExtractPlugin.loader : 'style-loader',
          'css-loader',
          'sass-loader'
        ]
      },
      {
        test: /\.md$/,
        use: [
          {
            loader: 'vue-loader',
            options: {
              compilerOptions: {
                preserveWhitespace: false
              }
            }
          },
          {
            loader: path.resolve(__dirname, './md-loader/index.js')
          }
        ]
      },
      {
        test: /\.(svg|otf|ttf|woff2?|eot|gif|png|jpe?g)(\?\S*)?$/,
        loader: 'url-loader',
        // todo: 这种写法有待调整
        query: {
          limit: 10000,
          name: path.posix.join('static', '[name].[hash:7].[ext]')
        }
      }
    ]
  },
  plugins: [
    // HMR 插件
    new webpack.HotModuleReplacementPlugin(),
    new HtmlWebpackPlugin({
      // 指定生成的文件所依赖文件模板
      template: './examples/index.tpl',
      filename: './index.html',
      favicon: './examples/favicon.ico'
    }),
    new CopyWebpackPlugin([
      // from: copy 文件路径；to: 默认 Output.path
      { from: 'examples/versions.json' }
    ]),
    new ProgressBarPlugin(),
    new VueLoaderPlugin(),
    // 配置的全局常量
    new webpack.DefinePlugin({
      'process.env.FAAS_ENV': JSON.stringify(process.env.FAAS_ENV)
    }),
    // 用于迁移
    new webpack.LoaderOptionsPlugin({
      vue: {
        compilerOptions: {
          preserveWhitespace: false
        }
      }
    })
  ],
  optimization: {
    minimizer: []
  },
  // 生成 SourceMap
  devtool: '#eval-source-map'
};

// 以下配置，仅在生产环境有效
if (isProd) {
  // 依赖包外部引用配置
  webpackConfig.externals = {
    vue: 'Vue',
    'vue-router': 'VueRouter',
    'highlight.js': 'hljs'
  };
  webpackConfig.plugins.push(
    // CSS 文件打包抽取
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash:7].css'
    })
  );
  webpackConfig.optimization.minimizer.push(
    new UglifyJsPlugin({
      // 开启文件缓存
      cache: true,
      // 开启并行处理提高构建速度
      parallel: true,
      // 是否生成 SourceMap
      sourceMap: false
    }),
    new OptimizeCSSAssetsPlugin({})
  );
  // 存在多入口文件打包的时候，出现了重复引用第三方库的问题
  // 使用 splitChunks 抽取公共模块
  // https://webpack.js.org/configuration/optimization/#optimizationsplitchunks
  webpackConfig.optimization.splitChunks = {
    // cacheGroups 可以继承或覆盖 splitChunks.* 的任何选项
    cacheGroups: {
      vendor: {
        // 匹配规则，说明要匹配的项
        test: /\/src\//,
        name: 'element-ui',
        // 将选择哪些 chunk 进行优化
        // 当提供一个字符串时，有效值为：all, async, initial。
        // 设置为 all 会特别强大，这意味着 chunk 可以在异步和非异步模块 chunk 之间共享。
        chunks: 'all'
      }
    }
  };
  // 是否生成 SourceMap
  webpackConfig.devtool = false;
}

module.exports = webpackConfig;
```

### `build/webpack.extension.js`

用于构建名为 `Element Theme Roller` 的 Chrome 插件项目，复用大部分 `webpack.demo.js` 打包配置。`npm run deploy:extension` 用于项目生产发布；`npm run dev:extension` 用于开发调试。

- 调用命令：`webpack --config build/webpack.extension.js`。
- 入口文件：`examples/extension/src/background.js` 和 `examples/extension/src/entry.js`。
- 输出文件：构建内容输出至 `examples/extension/dist` 目录下。生成文件 `background.js` 和 `entry.js`，复制文件 `icon.png` 和 `manifest.json`。

