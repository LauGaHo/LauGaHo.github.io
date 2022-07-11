# ElementUI工程化解析(一)

## ElementUI 源码目录结构

```
|-- build // 工程化构建脚本和配置
	|-- bin // 项目文件构建
	|-- md-loader // 自定义 markdown-loader，用于组件文档展示
|-- examples // 存放组件实例
|-- lib // 构建后生成的文件，发布到 npm 包
|-- packages // 存放组件源码和主题样式
|-- src // 存放入口文件和各种工具文件
	|-- directives // 指令
	|-- locale // 国际化功能
	|-- mixins // 混入方法
	|-- transition // 动画
	|-- utils // 工具方法
|-- test // 存放单元测试文件
|-- types // 存放声明文件，方便引入 TypeScript 项目中，需要在 package.json 中指定 typing 属性的值为声明的入口文件才能生效
|-- components.json // 完整组件清单，标注了组件的文件路径，方便 webpack 打包时获取组件的文件路径
|-- .travis.yml // 持续集成的配置文件，代码提交时，根据该文件执行对应脚本
|-- CHANGELOG.XX.md // 更新日志，4个不同语言版本
|-- FAQ.md // 对常见问题的解答
|-- LICENSE // 开源许可证
|-- Makefile // 通过 make new 创建组件目录结构，包含测试代码、入口文件、文档。make 是一个工程化编译工具
|-- package.json // 项目所需要的各种模块及配置信息
```

## package.json

项目中的 `package.json` 大致分为以下几类：

- 必备属性：`name`、`version`
- 描述信息：`description`、`keywords`、`homepage`、`repository`、`bugs`
- npm 脚本：`script`
- 依赖：`dependencies`、`devDependencies`、`peerDependencis`
- 协议：`license`
- 目录文件相关：`main`、`files`、`typings`、`faas`、`unpkg`、`style`

### 必备属性

`name` 和 `version` 是必须的属性，这两个属性组成一个 npm 模块的唯一标识。

`name` 是一个包的唯一标识，包名不能够重复，`npm view packageName` 查看包名是否被占用，如下图片所示：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/x4Z1ai.png)

### 依赖

根据依赖的配置信息：`dependencies`、`devDependencies`、`peerDependencies` 的配置信息，执行 `npm install || yarn || pnpm install` 就可以自动下载所需的模块，也就是项目所需的运行和开发环境。

### npm 脚本

指定了运行脚本命令的 npm 命令行缩写，覆盖整个项目的声明周期。

### 描述信息

记录项目相关的信息。

### 目录文件相关

#### `main`

`main` 属性指定了程序的主入口文件，当在应用程序中导入该软件包时，应用程序会在该位置搜索模块的导出。在代码中引入整个 Element 如下代码所示：`import ElementUI from 'element-ui'`，实际上引入的就是 `lib/element-ui.common.js` 中暴露出去的模块。

```json
{
  "main": "lib/element-ui.common.js",
}
```

#### `files`

`files` 属性描述 `npm publish` 后推送到 npm 服务器的文件列表，指定文件夹的话，则文件夹中的所有内容都会包含进来。也可以通过配置 `.npmignore` 文件来忽略文件上传。

```json
{
  "files": [
    "lib",
    "src",
    "packages",
    "types"
  ],
}
```

#### `typings`

`typings` 属性指定针对 TypeScript 的声明文件入口。

```json
{
  "typings": "types/index.d.ts"
}
```

#### `style`

`style` 属性指定了样式入口文件。

```json
{
    "style": "lib/theme-chalk/index.css",
}
```

#### `unpkg`

`unpkg` 是前端常用的公共 CDN，它通过 URL 语法就可以访问 npm 上任何包的任何文件

```
unpkg.com/:package@:version/:file
```

当把包发布到 npm 上时，不仅可以在 NodeJS 环境中使用，也可以通过 unpkg 获取在浏览器环境执行，不过需要符合 UMD 规范。

```json
{
  "unpkg": "lib/index.js"
}
```

`main` 属性值 `lib/element-ui.common.js` 是 CommonJS 规范，由 `build/webpack.common.js` 打包生成。`unpkg` 属性值 `lib/index.js` 是 UMD 规范，由 `build/webpack.conf.js` 打包生成。关于打包模块之后会讲解。

```javascript
// pkg 指 package.json
// 定义了 unpkg 属性时
const url = "https://unpkg.com/:package@:latestVersion/[pkg.unpkg]"

// 未定义 unpkg 属性时，将回退到 main 属性
const url = "https://unpkg.com/:package@:latestVersion/[pkg.main]"
```

设置了 `unpkg` 属性后，访问 `unpkg.com/element-ui` 按照规则将自动访问 `unpkg.com/element-ui/lib/index.js`。

若浏览器环境引入组件，只需要通过 `unpkg.com/element-ui` 获取到资源，在页面上引入对应的 JS 文件和 CSS 文件即可开始使用。CSS 文件路径是 `style` 属性值，JS 文件路径是基于 UMD 规范打包文件的路径—`unpkg` 的属性值。

```html
<!-- 引入样式 -->
<link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-chalk/index.css">
<!-- 引入组件库 -->
<script src="https://unpkg.com/element-ui/lib/index.js"></script>
```

#### `faas`

用于 `faas deploy` 配置，npm 脚本的 `pub` 命令存在 `sh build/deploy-faas.sh` 调用，用于站点 `element.eleme.io` 的发布部署。

## npm 脚本

`scripts` 属性指定了运行脚本命令的 npm 命令行缩写，各个脚本可以互相结合使用，这些脚本覆盖整个项目的声明周期 (开发、测试、打包、部署)。

> 注意事项：
>
> - 通配符：由于 npm 脚本就是 Shell 脚本，所以可以使用 Shell 通配符
>
>   ```json
>   "lint": "jshint *.js"
>   "lint": "jshint **/*.js"
>   ```
>
>   上方代码中，`*` 标识任意文件名，`**` 表示任意一层子目录
>
> - 执行顺序：如果 npm 脚本中需要执行多个任务，需要明确其执行顺序。如果是 **并行执行** (即同时的平行执行)，使用 `&` 符号。
>
>   ```shell
>   npm run script1.js & npm run script2.js
>   ```
>
>   如果是 **继发执行** (即只有前一个任务成功，才执行下一个任务)，使用 `&&` 符号。
>
>   ```shell
>   npm run script1.js && npm run script2.js
>   ```

`element` 定义了很多脚本，按照用途可以分为以下这些分类：

- 项目基础
- 文件构建
- 项目开发
- 发布部署
- 项目测试

脚本命令调用 `build` 目录中众多文件，自动化完成大量重复性工作，从而提高效率。

### 项目基础

#### `npm run bootstrap`

```json
"bootstrap": "yarn || npm i"
```

自动下载项目所需要的模块，官方推荐使用 `yarn`。

#### `npm run clean`

```json
"clean": "rimraf lib && rimraf packages/*/lib && rimraf test/**/coverage"
```

清除打包、测试生成的目录及文件，主要有 `lib` 目录、`test/unit/coverage` 目录以及 `packages/theme-chalk/lib` 目录。

> 需要安装 `rimraf` 包，用于递归删除目录所有文件。

#### `npm run lint` 代码质量检查

```json
"lint": "eslint src/**/* test/**/* packages/**/* build/**/* --quiet"
```

基于 `.eslintrc` 和 `.eslintignore` 文件配置，调用 `eslint` 检测代码规范，`--quiet` 参数允许报告错误，禁止报告警告。

项目使用自己封装的规则配置 `eslint-config-elemefe`，如下显示：

```json
{
  "extends": "elemefe"
}
```

### 文件构建

#### `npm run i18n`

```json
"i18n": "node build/bin/i18n.js"
```

执行 `build/bin/i18n.js` 基于 `examples/i18n/page.json` 页面多语言配置和 `examples/pages/templates` 目录下的所有模板文件，生成 `zh-CN`、`en-US`、`es`、`fr-FR` 等四种语言的网站 `.vue` 文件。

#### `npm run build:file`

```javascript
"build:file": "node build/bin/iconInit.js & node build/bin/build-entry.js & node build/bin/i18n.js & node build/bin/version.js"
```

该命令主要用于文件的自动生成，多个任务间是并行执行：

- 执行 `build/bin/iconInit.js` 生成 `examples/icon.json` 图标集合文件。
- 执行 `build/bin/build-entry.js` 生成 `src/index.js` 组件库入口文件。
- 执行 `build/bin/i18n.js` 生成官网的多语言网站文件。
- 执行 `build/bin/version.js` 生成 `examples/version.json` 记录项目版本信息，用于网站版头部导航版本切换。

#### `npm run build:theme`

```json
"build:theme": "node build/bin/gen-cssfile && gulp build --gulpfile packages/theme-chalk/gulpfile.js && cp-cli packages/theme-chalk/lib lib/chalk"
```

该命令主要用于项目的主题和样式生成。

- 执行 `build/bin/gen-cssfile` 生成 `packages/theme-chalk/index.scss` 样式总入口文件。全量引入组件时，引用该样式如下：`import 'packages/theme-chalk/src/index.scss'`。
- 采用 `gulp` 进行样式构建，将 `packages/theme-chalk/src` 下的 `scss` 文件转换成 `css` 文件，输出到 `packages/theme-chalk/src/lib` 目录下；将 `packages/theme-chalk/src/fonts` 下的字体文件压缩处理，输出到 `packages/theme-chalk/src/lib/fonts` 目录下。
- 将构建内容 `packages/theme-chalk/lib` 拷贝 `lib/theme-chalk` 下。前面 `style` 属性配置的路径文件 `lib/theme-chalk/index.css` 就是这样生成的。

> 需要安装 `cp-cli` 包，用于文件和文件夹的复制，无需担心跨平台的问题。

#### `npm run build:utils`

```json
"build:utils": "cross-env BABEL_ENV=utils babel src --out-dir lib --ignore src/index.js"
```

该命令作用把 `src` 目录下除了 `src/index.js` 入口文件外的其他文件通过 `babel` 转译后，输出至 `lib` 文件夹下。

> 需要安装 `cross-env` 包，是一款运行跨平台设置和使用环境变量的脚本，不同平台使用唯一指令，无需担心跨平台问题。

#### `npm run build:umd`

```json
"build:umd": "node build/bin/build-locale.js"
```

该命令作用是执行 `build/bin/build-locale.js` 通过 `babel` 处理 `src/locale/lang` 目录下的文件，生成 `umd` 格式的文件，输出到 `lib/umd/locale` 目录下。

### 项目开发

#### `npm run dev`

```json
"dev": "npm run bootstrap && npm build:file && cross-env NODE_ENV=development webpack-dev-server --config build/webpack.demo.js & node build/bin/template.js"
```

该命令用于运行组件库的本地开发环境：

- 执行命令 `npm run bootstrap` 安装项目所需的依赖。
- 执行命令 `npm run build:file` 生成各种入口文件。
- `webpack-dev-server` 提供一个本地服务并运行项目网站 (打包规则配置文件 `build/webpack.demo.js`)；同时执行 `build/bin/template.js` 文件启动监听 `examples/pages/template` 目录下的模板文件，若内容发生变化，则重新生成网站文件。`webpack-dev-server` 具有 `live reloading` 功能，网站内容会实时重新加载。

#### `npm run dev:play`

```json
"dev:play": "npm run build:file && cross-env NODE_ENV=development PLAY_ENV=true webpack-dev-server --config build/webpack.demo.js"
```

该命令用于组件库开发得的功能展示：

- 执行 `npm run build:file` 生成对应的入口文件。
- 因为配置了环境变量 `NODE_ENV=development PLAY_ENV=true`，可以在 `build/webpack.demo.js` 打包文件中看到入口文件 `example/play.js`，`play.js` 引用 `examples/play/index.vue`，可以引入组件库任意一个组件用于功能展示。

### 发布部署

#### `npm run deploy:build`

```json
"deploy:build": "npm run build && cross-env NODE_ENV=production webpack --config build/webpack.demo.js && ehco element.eleme.io>>examples/element-ui/CNAME"
```

该命令作用主要用于打包构建项目官网内容，为网站部署做准备：

- 执行命令 `npm run build:file` 生成对应的入口文件。
- `webpack --config build/webpack.demo.js` 基于 `production` 模式，打包生成内容输出至 `examples/element-ui/` 目录下。
- `echo element.eleme.io>>examples/element-ui/CNAME` 文件中写入 `element.eleme.io`。

#### `npm run deploy:extension`

```json
"deploy:extension": "cross-env NODE_ENV=production webpack --config build/webpack.extension.js"
```

该命令作用是打包构建主题编辑器的 Chrome 插件项目：`Element Theme Roller`。基于 `production` 模式打包生成内容输出到 `examples/extension` 目录下。

使用该插件可以自定义全局变量和组件所有设计标记，并实时预览新主题并基于新主题生成完整的样式包，以供下载。

#### `npm run dist`

```json
"dist": "npm run clean && npm run build:file && npm run lint && webpack --config build/webpack.conf.js && webpack --config build/webpack.common.js && webpack --config build/webpack.component.js && npm run build:utils &&npm run build:umd && npm run build:theme"
```

该命令：

- `npm run clean` 清除打包、测试等文件目录。

- `npm run build:file` 生成对应的入口文件。

- `npm run lint` 检查代码规范。

- 执行打包 `webpack --config build/webpack.conf.js`，入口文件 `src/index.js` 以 UMD 格式输出到 `lib/index.js`。

- 执行打包 `webpack --config build/webpack.common.js`，入口文件 `src/index.js` 以 CommonJS2 格式输出到 `lib/element-ui.common.js`。

- 执行打包 `webpack --config build/webpack.component.js`，入口文件 `components.json`，将 `packages` 目录下的组件，以 CommonJS2 格式分别输出到 `lib` 目录下，用于按需引入。

- `npm run build:utils` 将 `src` 目录下除了 `index.js` 文件之外的文件通过 Babel 转译后生成对应的 JS 文件，并存放在 `lib` 目录下。

  > 注意：`src` 目录下，一般都是存放 `directives`、`locale`、`mixins` 的内容，并没有组件的内容，组件的定义都在 `packages` 文件下。

- `npm run build:umd` 通过 Babel 处理 `src/locale/lang` 目录下的文件，生成 UMD 格式的文件，输出到 `lib/umd/locale` 目录下。

- `npm run build:theme` 生成样式总入口文件，并将 `.scss` 编译成 `.css` 文件，输出到 `packages/theme-chalk/src/lib` 目录下，最后将构建内容 `packages/theme-chalk/lib` 拷贝到 `lib/theme-chalk` 下。