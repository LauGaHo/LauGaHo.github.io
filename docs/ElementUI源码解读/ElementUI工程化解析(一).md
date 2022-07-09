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

