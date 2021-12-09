# Webpack

## 自动引入资源

- **使用 HtmlWebpackPlugin**

  1. **首先安装插件**

     ```shell
     npm install html-webpack-plugin --save-dev
     ```

     **并且调整 `webpack.config.js` 文件：**

     ```javascript
     module.exports = {
       // ...
       plugins: [
         // 实例化 html-webpack-plugin 插件
         new HtmlWebpackPlugin()
       ]
     }
     ```

     **执行打包命令：**

     ```shell
     npx webpack
     ```

     **打包自动生成的 `index.html` 的内容如下：**

     ```html
     <!DOCTYPE html>
     <html>
       <head>
         <meta charset="utf-8">
         <title>Webpack App</title>
         <meta name="viewport" content="width=device-width, initial-
         scale=1">
         <script defer src="bundle.js"></script>
       </head>
       <body>
       </body>
     </html>
     ```

     **此时生成的 html 文件是新创建的 html 文件，如果我们已经有了对应的 html 文件，想让 webpack 在我们已有的 html 文件的基础上引入我们的编译结果，那么需要我们调整一下 `webpack.config.js` 文件如下：**

     ```javascript
     plugins: [
       // 实例化 html-webpack-plugin 插件 
       new HtmlWebpackPlugin({
         // 打包生成的文件的模板
         template: './index.html', 
         // 打包生成的文件名称。默认为index.html 
         filename: 'app.html', 
         // 设置所有资源文件注入模板的位置。可以设置的值true|'head'|'body'|false，默认值为 true 
         inject: 'body'
     	}) 
     ]
     ```

     **这时就代表了 webpack 在编译结束后，会在我们给定的 html 文件当中引入对应的编译结果，并且将该 html 命名为 `app.html`，并且还会将脚本的引入位置从 `<head>` 迁移到 `<body>` 中，如下所示：**

     ```html
     <!DOCTYPE html>
     <html lang="en">
       <head>
         <meta charset="UTF-8">
         <meta http-equiv="X-UA-Compatible" content="IE=edge">
         <meta name="viewport" content="width=device-width, initial-
         scale=1.0">
       </head>
       <body>
      		<script defer src="bundle.js"></script>
       </body> 
     </html>
     ```



## 清理 dist 文件夹

- **每次重新编译的时候，dist 文件夹中都还是包含着上一次编译产生的残留文件，所以我们需要在每次编译之前将 dist 文件夹中的所有文件清空**

  ```javascript
  module.exports = {
    
    //...
    output: {
      //...
      // 打包前清空 dist 文件夹
      clean: true
    },
    //...
  }
  ```

  **这个时候再次打包就会清空 dist 文件夹之后再执行打包操作**



## 搭建开发环境

- **截止目前，我们只能通过复制 `dist/index.html` 完整物理路径到浏览器地址栏进行访问页面，但是可以搭建一个开发环境，使我们开发体验变得轻松**

- **设置 mode 选项开始之前，我们需要将 `mode` 设置为 `development`**

  ```javascript
  module.exports = {
    //...
    // 开发模式
    mode: 'development',
    //...
  }
  ```

- **使用 Source Map。当使用 webpack 打包源代码时，可能会很难追踪到 error 和 warning 在源代码中的原始位置，例如将三个源文件 (`a.js`, `b.js`, `c.js`) 打包到一个 `bundle.js` 中，而其中一个源文件包含一个错误，那么堆栈跟踪就会直接指向到 `bundle.js` 。你可能需要准确知道错误来自于哪个源文件，所以这种提示通常没啥用，为了更好追踪代码的 error 和 warning，JavaScript 提供了 Source maps 功能，可以将编译后的代码映射会原始源代码，帮助你准确定位出错位置。在本篇我们将使用 `inline-source-map` 选项：**

  ```javascript
  module.exports = {
    //...
    // 开发模式下追踪代码
    devtool: 'inline-source-map',
    //...
  }
  ```

- **使用 watch mode (观察模式)，每次编译代码，都需要手动运行 `npx webpack` 会显得很麻烦，我们可以在 webpack 启动的时候添加一个 "watch" 参数。如果其中一个文件被更新了，代码将被重新编译，所以就不必手动去运行 `npx webpack` 整个构建，执行了该命令之后，命令行的光标就会停留在尾部，监测文件的改变，如果修改了文件，就会重新编译代码，现在只剩下唯一的缺点，那就是为了看到修改后的效果，你需要手动刷新浏览器**

  ```shell
  npx webpack --watch
  ```

- **使用 `webpack-dev-server`，`webpack-dev-server` 为你提供了一个基本的 web-server，并且具有 live-reloading (实时加载) 功能，先进行安装**

  ```shell
  npm install --save-dev webpack-dev-server
  ```

  **修改配置文件，告知 dev-server，从什么位置开始查找文件：**

  ```javascript
  module.exports = {
    //...
    // dev-server
    devServer: {
      static: './dist'
    }
  }
  ```

  **以上告知 `webpack-dev-server`，将 dist 目录下的文件作为 web 服务器的根目录**

  > **webpack-dev-server 在编译之后不会写入到任何的输出文件。而是将 bundle 文件保留在内存中，然后将它们 serve 到 server 中，就好像他们是挂载在 server 根路径上的真实文件一样**



### 资源模块

- **资源模块 (asset module) 是一种模块类型，它允许我们应用 webpack 来打包其他资源文件 (如字体、图标等)**

- **资源模块类型 (asset module type)，包含 4 中新的模块类型：**

  - `asset/resource` ：输出一个单独的文件，并导出 URL
  - `asset/inline` ：导出一个资源的 data URI
  - `asset/source` ：导出资源的源代码
  - `asset` ：在导出一个 data URI 和输出一个单独的文件之间自动选择

- **Resource 资源**	

  - **修改 `webpack.config.js` 配置：**

    ```javascript
    module.exports = {
      //...
      // 配置资源文件
      module: {
        rules: [{
          test: /\.png$/,
          type: 'asset/resource'
        }]
      }
      //...
    }
    ```

  - **在入口文件中进行引入操作：**

    ```javascript
    // 导入函数模块
    import helloWorld from './hello-world.js' import imgsrc from './assets/img-1.png'
    helloWorld()
    const img = document.createElement('img')
    img.src = imgsrc
    document.body.appendChild(img)
    ```

  - **执行打包命令：**

    ```shell
    npx webpack
    ```

  - **这个时候，就会发现对应的图片文件 (.png) 就已经打包到 dist 目录下了：**

    ![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/Is1BFE.png)

  - **我们还可以通过修改配置达到自定义输出文件名的效果，如下：**

    ```javascript
    module.exports = {
      //...
      output: {
        //...
        assetModuleFilename: 'images/[contenthash][ext][query]'
      }
      //...
    }
    ```

  - **执行编译，就会得到这个目录结果：**

    ![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/DuVpPT.png)

  - **还有一种自定义输出文件名的方式是，将某些资源输出到指定目录，修改配置：**

    ```javascript
    module.exports = {
      //...
      output: {
        //...
        // 配置资源文件
        module: {
          rules: [
            {
              test: /\.png$/,
              type: 'asset/resource',
              // 优先级高于 assetModuleFilename
              generator: {
                filename: 'images/[contenthash][ext][query]'
              }
            }
          ]
        }
      }
      //...
    }
    ```

    **执行编译，输出结果跟 `assetModuleFilename` 设置一样**

    

- **inline 资源**

  - **修改 `webpack.config.js` 配置**

    ```javascript
    module.exports = {
      //...
      // 配置资源文件
      module: {
        rules: [
          {
            test: /\.svg$/,
            type: 'asset/inline'
          }
        ]
      }
      //...
    }
    ```

  - **执行打包命令：**

    ```shell
    npx webpack serve --open
    ```

    **这个时候就可以看到，`svg` 文件将作为 data URI 也就是 base64 注入到 bundle 中**

    ![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/eaZY17.png)

 - **source 资源**

   **source 资源，导出资源的源代码。修改配置文件，添加：**

   ```javascript
   module.exports = {
     //...
     //配置资源文件
     module: {
       rules: [
         //...
         {
           test: /\.txt$/,
           type: 'assetsource'
         }
       ]
     }
   }
   ```

   **在 assets 里创建一个 `example.txt` 文件：**

   ```txt
   hell webpack
   ```

   **在入口文件引入一个 `.txt` 文件，添加内容：**

   ```javascript
   // 导入函数模块
   //...
   import exampleText from './assets/example.txt'
   //...
   const block = document.createElement('div')
   block.style.cssText = `width: 200px; height: 200px; background: aliceblue`
   block.textContent = exampleText
   document.body.appendChild(block)
   //...
   ```

- **通用资源类型**

  **通用资源类型 `asset`，在导出一个 data URI 和发送一个单独的文件之间自动选择。修改配置文件：**

  ```javascript
  module: {
    rules: [
      {
        test: /\.jpg$/,
     		type: 'asset',
      }
    ]
  }
  ```

  **现在，webpack 将按照默认条件，自动在 `resource` 和 `inline` 之间进行选择：小于 8kb 的文件，将会视为 `inline` 模块类型，否则会被视为 `resource` 模块类型。我们可以通过在 webpack 配置的 module rules 层级中，设置 `Rule.parser.dataUrlCondition.maxSize` 选项来修改此条件：**

  ```javascript
  module.exports = {
    //...
    module: {
      rules: [
        //...
        {
          test: /\.jpg$/,
          type: 'asset',
          parser: {
            dataUrlCondition: {
              maxSize: 4 * 1024 // 4kb
            }
          }
        }
      ]
    }
  }
  ```

  

## 管理资源

**在上一章中，我们通过讲解了四种资源模块引入外部资源。除了资源模块，我们还可以通过 `loader` 引入其他类型的文件。**

- **什么是 loader**

  **webpack 只能理解 JavaScript 和 JSON 文件，这是 webpack 开箱可用的自带功能。loader 让 webpack 能够去处理其他类型的文件，并将他们转换为有效的模块 (所谓模块就是文件，一个文件就是一个模块) ，以供应用程序使用，以及被添加到依赖图中。**

  **在 webpack 的配置中，loader 有两个属性：**

  - **`test` 属性，识别出哪些文件需要被转换**

  - **`use` 属性，定义在进行转换时，应该使用哪个 loader**

    ```javascript
    const path = require('path');
    
    module.exports = {
      output: {
        filename: 'my-first-webpack.bundle.js',
      },
      module: {
        rules: [
          {
            test: /\.txt$/,
            use: 'raw-loader'
          }
        ]
      }
    }
    ```

    **以上配置中，对一个单独的 `module` 对象定义了 `rules` 属性，里面包含两个必须的属性：`test` 和 `use` 。这告诉 webpack 编译器如下信息：**

    > **"嘿，webpack 编译器，当你如果碰到了 [在 `require()/import` 语句中解析为 `'txt'` 的路径] 时，在你对他打包前，先 use (使用) `rau-loader` 转换一下"**



- **加载 CSS**

  **为了在 JavaScript 模块中 `import` 一个 CSS 文件，你需要安装 `style-loader` 和 `css-loader` ，并在 module 配置中添加这些 loader：**

  ```shell
  npm install --save-dev style-loader css-loader
  ```

  **修改配置文件：**

  ```javascript
  module.exports = {
    //...
    // 配置资源文件
    module: {
      rules: [
        //...
        {
          test: /\.css$/,
          use: ['style-loader', 'css-loader'],
        }
      ]
    }
    //...
  }
  ```

  **模块 loader 可以链式调用。链中的每个 loader 都将对资源进行转换。链会逆序执行，第一个 loader 将其结果 (被转换后的资源) 传递给下一个 loader，以此类推，最后，webpack 期望链中的最后的 loader 返回 JavaScript。**

  **应保证 loader 的先后顺序：`style-loader` 在前，而 `css-loader` 在后，如果不遵守此约定，webpack 可能会抛出错误。webpack 根据正则表达式，来确定应该查找哪些文件，并将其提供给指定的 loader。在这个示例中，所有以 `.css` 结尾的文件都将被提供给 `style-loader` 和 `css-loader`。**

  **这使你可以在依赖此样式的 js 文件中 `import './style.css'`。现在，在此模块执行过程中，含有 CSS 字符串的 `<style>` 标签，将被插入到 html 文件的 `<head>` 标签中。**

  **现有的 loader 可以支持其他的 CSS 预处理器，例如像 LESS、SCSS、SASS，这需要我们安装对应的 loader：**

  ```shell
  npm install less less-loader --save-dev
  ```

  **修改配置文件：**

  ```javascript
  module.exports = {
    //...
    // 配置资源文件
    module: {
      rules: [
        //...
        {
          test: /\.less$/,
          use: ['style-loader', 'css-loader', 'less-loader']
        }
      ]
    }
  }
  ```

  **相比使用 CSS 文件，只需要在 use 配置项中的数组添加一项 `less-loader` 即可。**



- **抽离和压缩 CSS**

  **在多数情况下，我们也可以进行压缩 CSS，以便在生产环境中节省加载时间，同时还将 CSS 文件抽离成一个单独的文件。实现这个功能，需要使用 `mini-css-extract-plugin` 这个插件来帮忙。安装插件：**

  ```shell
  npm install mini-css-extract-plugin --save -dev
  ```

  **本插件会将 CSS 提取到单独的文件中，为每个包含 CSS 的 JS 文件创建一个 CSS 文件，并且支持 CSS 和 SourceMaps 的按需加载**

  **本插件基于 webpack 5 的新特性构建，并且需要 webpack 5 才能正常工作。**

  **安装完之后将 loader 和 plugin 添加到 webpack 配置文件中：**

  ```javascript
  const MiniCssExtractPlugin = require("mini-css-extract-plugin")
  
  module.exports = {
    module: {
      rules: [
        {
          test: /\.css$/,
          use: [MiniCssExtractPlugin.loader, 'css-loader'],
        },
        //...
      ]
    }
    //...
  }
  ```

  **单独的 `mini-css-extract-plugin` 插件不会将这些 CSS 加载到页面中。这里的 `html-webpack-plugin` 帮助我们自动生成 `link` 标签或者在创建 `index.html` 文件时使用 `link` 标签，输出的结果如下：**

  ```html
  <!DOCTYPE html>
  <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta http-equiv="X-UA-Compatible" content="IE=edge">
      <meta name="viewport" content="width=device-width, initial- scale=1.0">
      <title>千锋大前端教研院-Webpack5学习指南</title>
      <link href="main.css" rel="stylesheet">
    </head>
    <body>
    	<script defer src="bundle.js"></script>
    </body>
  </html>
  ```

  **这时，`link` 标签已经生成出来了，把我们打包好的 `main.css` 文件加载进来。我们发现，`main.css` 文件被打包抽离到 `dist` 根目录下，我们可以通过配置，将其打包到一个单独的文件夹中，修改配置文件：**

  ```javascript
  const MiniCssExtractPlugin = require("mini-css-extract-plugin")
  
  module.exports = {
    //...
    plugins: [
      new MiniCssExtractPlugin({
        filename: 'styles/[contenthash].css'
      })
    ]
    //...
  }
  ```

  **再次执行编译：**

  ```shell
  npx webpack
  ```

  **查看打包完成后的目录和文件：**

  ![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/VFQs1y.png)

  **查看对应的 CSS 文件：**

  ```css
  /*!**********************************************************
  ********!*\
  !*** css ../node_modules/css-
  loader/dist/cjs.js!./src/style.css ***!
  \************************************************************
  ******/
  .hello {
      background-color: #f9efd4;
  }
  /*# sourceMappingURL=data:application/json;charset=utf-
  8;base64,eyJ2ZXJzaW9uIjozLCJmaWxlIjoic3R5bGVzLzRhMDUzMTlkYWM5
  MDJlMjc5ODM5LmNzcyIsIm1hcHBpbmdzIjoiOzs7QUFBQTtFQUNFLHlCQUF5Q
  jtBQUMzQixDIiwic291cmNlcyI6WyJ3ZWJwYWNrOi8vLy4vc3JjL3N0eWxlLm
  NzcyJdLCJzb3VyY2VzQ29udGVudCI6WyIuaGVsbG8ge1xuICBiYWNrZ3JvdW5
  kLWNvbG9yOiAjZjllZmQ0O1xufSJdLCJuYW1lcyI6W10sInNvdXJjZVJvb3Qi
  OiIifQ==*/
  ```

  

  **此时发现 CSS 文件没有压缩和优化，为了压缩输出文件，需要使用类似于 `css-minimizer-webpack-plugin` 这样的插件。安装插件：**

  ```shell
  npm install css-minimizer-webpack-plugin --save-dev
  ```

  **配置插件**

  ```javascript
  const CssMinimizerPlugin = require('css-minimizer-webpack-plugin')
  
  module.exports = {
    // 生产模式
    mode: 'production',
    
    // 优化配置
    optimization: {
      minimizer: [
        new CssMinimizerPlugin(),
      ]
    }
  }
  ```

  **再次执行编译：**

  ```shell
  npx webpack
  ```

  **对应的 CSS 文件变成如下：**

  ```css
  .hello{background-color:#f9efd4}
  /*# sourceMappingURL=data:application/json;charset=utf-
  8;base64,eyJ2ZXJzaW9uIjozLCJmaWxlIjoic3R5bGVzLzQ3ZDc2ZDUzNmM2
  NmVmYWY3YTU1LmNzcyIsIm1hcHBpbmdzIjoiQUFBQSxPQUNFLHdCQUNGIiwic
  291cmNlcyI6WyJ3ZWJwYWNrOi8vLy4vc3JjL3N0eWxlLmNzcyJdLCJzb3VyY2
  VzQ29udGVudCI6WyIuaGVsbG8ge1xuICBiYWNrZ3JvdW5kLWNvbG9yOiAjZjl
  lZmQ0O1xufSJdLCJuYW1lcyI6W10sInNvdXJjZVJvb3QiOiIifQ==*/
  ```

  **此时 CSS 优化成功！**

- **加载 images 图像**

  **loader 能够对 CSS 文件进行编译，但是像 background 和 icon 这样的图像，在 webpack 5 中可以使用内置的 Asset Modules，我们可以轻松地将这些内容混入到我们的系统中，这里再补充一个知识点，在 CSS 文件中也可以直接引用文件，修改 `style.css` 和 `index.js` 文件：**

  ```css
  // style.css
  .block-bg {
    background-image: url(./assets/webpack-logo.svg);
  }
  ```

  ```javascript
  // index.js
  block.style.cssText = `width: 200px; height: 200px; background-color: #2b3a42`
  block.classList.add('block-bg')
  ```

- **加载 fonts 字体**

  **像字体这样的资源可以使用 Asset Modules 接收并加载任何文件，然后将其输出到构建目录。这就是说，我们可以将他们用于任何类型的文件，让我们修改 `webpack.config.js` 来处理字体文件：**

  ```javascript
  module.exports = {
    module: {
      rules: [
        {
          test: /\.(woff|woff2|eot|ttf|otf)$/i,
          type: 'asset/resource'
        }
      ]
  	}
  }
  ```



## 使用 babel-loader

**前面的章节里，我们应用 `less-loader` 编译过 less 文件，应用 `xml-loader` 编译过 xml 文件，那 js 文件如果需要将 es6 转换为 es5 的话，则需要对 js 文件进行编译，修改 `hello-world.js` 文件**

```javascript
function getString() {
return new Promise((resolve, reject) => {
   setTimeout(() => {
      resolve('Hello world~~~')
}, 2000) })
}
async function helloWorld() {
let string = await getString()
console.log(string)
}
// 导出函数模块
export default helloWorld
```

- **使用 babel-loader 方法如下：**

  **安装：**

  ```shell
  npm install babel-loader @babel/core @babel/preset-env
  ```

  - `babel-loader`：在 webpack 里应用 babel 解析 ES6 的桥梁
  - `@babel/core`：babel 核心模块
  - `@babel/preset-env`：babel 预设，一组 babel 插件的集合

  **在 webpack 配置中，需要将 `babel-loader` 添加到 `module` 列表中，就像下面这样：**

  ```javascript
  module.exports = {
    //...
    module: {
      rules: [
        {
          test: /\.js$/,
          exclude: /node_modules/,
          use: {
            loader: 'babel-loader',
            options: {
              preset: ['@babel/preset-env']
            }
          }
        }
      ]
    }
  }
  ```

  **执行编译：**

  ```shell
  npx webpack
  ```

  **查看对应的编译后的文件：**

  ```javascript
  /***/
  "./src/hello-world.js":
  /*!****************************!*\
  !*** ./src/hello-world.js ***!
  \****************************/
  /***/
  ((__unused_webpack_module, __webpack_exports__,
  __webpack_require__) => {
  "use strict";
  __webpack_require__.r(__webpack_exports__);
  /* harmony export */
  __webpack_require__.d(__webpack_exports__, {
  /* harmony export */
  "default": () => (__WEBPACK_DEFAULT_EXPORT__)
  /* harmony export */
  });
  function asyncGeneratorStep(gen, resolve, reject, _next,
  _throw, key, arg) {
    try {
     var info = gen[key](arg);
     var value = info.value;
    } catch (error) {
     reject(error);
    	return;
    }
    if (info.done) {
     resolve(value);
    } else {
     Promise.resolve(value).then(_next, _throw);
    } 
  }
    
  function _asyncToGenerator(fn) {
    return function () {
     var self = this,
       args = arguments;
     return new Promise(function (resolve, reject) {
       var gen = fn.apply(self, args);
       function _next(value) {
         asyncGeneratorStep(gen, resolve, reject, _next, _throw,
    "next", value);
       }
       function _throw(err) {
         asyncGeneratorStep(gen, resolve, reject, _next, _throw,
    "throw", err);
         }
       _next(undefined);
     });
    }; 
  }
    
  function getString() {
    return new Promise(function (resolve, reject) {
     setTimeout(function () {
       resolve('Hello world~~~');
    	}, 2000); 
    });
  }
    
  function helloWorld() {
  	return _helloWorld.apply(this, arguments); 
  } // 导出函数模块
    
  function _helloWorld() {
    _helloWorld = _asyncToGenerator( /*#__PURE__*/
    regeneratorRuntime.mark(function _callee() {
     var string;
     return regeneratorRuntime.wrap(function _callee$(_context) {
       while (1) {
         switch (_context.prev = _context.next) {
           case 0:
             _context.next = 2;
             return getString();
           case 2:
             string = _context.sent;
             console.log(string);
           case 4:
           case "end":
             return _context.stop();
         }
       }
     }, _callee);
    }));
    return _helloWorld.apply(this, arguments);
  }
  /* harmony default export */
  const __WEBPACK_DEFAULT_EXPORT__ = (helloWorld);
    
  /***/
  }),
  ```

  **从编译完的结果可以看出，`async/await` 的 ES6 语法被 `babel` 编译了。**

  - **regeneratorRuntime 插件**

    **上面编译完之后，在浏览器打开，会显示有一个致命的错误**

    ![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/pd8DEw.png)

    **`regeneratorRuntime` 是 webpack 打包生成的全局辅助函数，由 babel 生成，用于兼容 `async/await` 的语法**

    **正确的做法是需要添加一下的插件和配置：**

    ```shell
    # 这个包中包含了regeneratorRuntime，运行时需要 
    npm install --save @babel/runtime
    # 这个插件会在需要regeneratorRuntime的地方自动require导包，编译时需要 
    npm install --save-dev @babel/plugin-transform-runtime
    ```

    **接着修改一下 babel 的配置：**

    ```javascript
    module.exports = {
      //...
      // 配置资源文件
      module: {
        rules: [
          {
            test: /\.js$/,
            exclude: /node_module/,
            use: {
              loader: 'babel-loader',
              options: {
                preset: ['@babel/preset-env'],
                plugins: [
                  [
                    '@babel/plugin-transform-runtime'
                  ]
                ]
              }
            }
          }
        ]
      }
    }
    ```

    **这个时候重新编译，打开浏览器，就可以成功运行了。**

