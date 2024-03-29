# ElementUI工程化解析(二)之自动化构建脚本

## 组件清单 `components.json`

`components.json` 是一份项目完整的组件清单，列出了组件的名称、文件路径、在项目文件构建、Webpack 等多处用到。

`components.json` 文件不是自动生成的，但其清单内容是自动更新的。当使用 `make` 命令创建新组件 `make new <component-name> [中文名]` 时自动更新组件清单，具体实现在 `build/bin/new.js` 中讲解。

```json
{
  "pagination": "./packages/pagination/index.js",
  "dialog": "./packages/dialog/index.js",
  "autocomplete": "./packages/autocomplete/index.js",
  "dropdown": "./packages/dropdown/index.js",
  "dropdown-menu": "./packages/dropdown-menu/index.js",
  "dropdown-item": "./packages/dropdown-item/index.js",
  ......
}

```

## 项目构建 `build/bin`

#### `build/bin/build-entry.js`

生成组件库入口文件 `src/index.js`。基于组件清单文件 `components.json` 结合字符串模板 `json-templater/string` 自动生成。当组件清单变更时不需要手动更新文件，只需要运行该文件自动生成最新文件覆盖更新。

```javascript
// 项目组件清单
var Components = require('../../components.json');
var fs = require('fs');
// 字符串模板
var render = require('json-templater/string');
// 转换成大驼峰命名工具函数
// ‘checkbox-button’ => ‘CheckboxButton’
var uppercamelcase = require('uppercamelcase');
var path = require('path');
var endOfLine = require('os').EOL;

// 文件输出路径 src/index.js
var OUTPUT_PATH = path.join(__dirname, '../../src/index.js');
// 组件导入模板
// 生成事例：import {{name}} from '../packages/{{pacakge}}/index.js;'
var IMPORT_TEMPLATE = 'import {{name}} from \'../packages/{{package}}/index.js\';';
// 大驼峰命名格式的组件名称
var INSTALL_COMPONENT_TEMPLATE = '  {{name}}';
// src/index.js 内容模板
var MAIN_TEMPLATE = `/* Automatically generated by './build/bin/build-entry.js' */

{{include}}
import locale from 'element-ui/src/locale';
import CollapseTransition from 'element-ui/src/transitions/collapse-transition';

const components = [
{{install}},
  CollapseTransition
];

const install = function(Vue, opts = {}) {
  locale.use(opts.locale);
  locale.i18n(opts.i18n);

  components.forEach(component => {
    Vue.component(component.name, component);
  });

  Vue.use(InfiniteScroll);
  Vue.use(Loading.directive);

  Vue.prototype.$ELEMENT = {
    size: opts.size || '',
    zIndex: opts.zIndex || 2000
  };

  Vue.prototype.$loading = Loading.service;
  Vue.prototype.$msgbox = MessageBox;
  Vue.prototype.$alert = MessageBox.alert;
  Vue.prototype.$confirm = MessageBox.confirm;
  Vue.prototype.$prompt = MessageBox.prompt;
  Vue.prototype.$notify = Notification;
  Vue.prototype.$message = Message;

};

/* istanbul ignore if */
if (typeof window !== 'undefined' && window.Vue) {
  install(window.Vue);
}

export default {
  version: '{{version}}',
  locale: locale.use,
  i18n: locale.i18n,
  install,
  CollapseTransition,
  Loading,
{{list}}
};
`;

delete Components.font;
// 得到所有的组件名称，形如：['input', 'checkbox-button', ...]
var ComponentNames = Object.keys(Components);

// 存放所有组件的 import 语句
var includeComponentTemplate = [];
// 存放所有组件的 install 语句
var installTemplate = [];
// 存放需要 export 的组件名字
var listTemplate = [];

// 遍历组件名字
ComponentNames.forEach(name => {
  // 将组件名字变成大驼峰形式
  var componentName = uppercamelcase(name);

  // 生成组件 import 语句，并将其放入 includeComponentTemplate 数组中
  includeComponentTemplate.push(render(IMPORT_TEMPLATE, {
    name: componentName,
    package: name
  }));

  // 形如 Loading、MessageBox、Message 等组件不需要全局注册，只需要 install
  if (['Loading', 'MessageBox', 'Notification', 'Message', 'InfiniteScroll'].indexOf(componentName) === -1) {
    installTemplate.push(render(INSTALL_COMPONENT_TEMPLATE, {
      name: componentName,
      component: name
    }));
  }

  // 如果组件不为 Loading，将其作为 export 中的一员进行导出
  if (componentName !== 'Loading') listTemplate.push(`  ${componentName}`);
});

// src/index.js 模板替换
var template = render(MAIN_TEMPLATE, {
  include: includeComponentTemplate.join(endOfLine),
  install: installTemplate.join(',' + endOfLine),
  version: process.env.VERSION || require('../../package.json').version,
  list: listTemplate.join(',' + endOfLine)
});

// 模板内容替换生成后，输出文件 src/index.js
fs.writeFileSync(OUTPUT_PATH, template);
// 返回处理进度
console.log('[build entry] DONE:', OUTPUT_PATH);
```

#### `build/bin/build-locale.js`

通过 Babel 处理 `src/locale/lang` 目录下的翻译文件，生成 UMD 格式文件，输出到 `lib/umd/locale` 目录下。

```javascript
var fs = require('fs');
var save = require('file-save');
var resolve = require('path').resolve;
var basename = require('path').basename;
// 本地化的翻译文件目录
var localePath = resolve(__dirname, '../../src/locale/lang');
// 获取目录下所有文件名称的数组对象
var fileList = fs.readdirSync(localePath);

// 转换函数
var transform = function(filename, name, cb) {
  require('babel-core').transformFile(resolve(localePath, filename), {
    plugins: [
      'add-module-exports',
      ['transform-es2015-modules-umd', {loose: true}]
    ],
    moduleId: name
  }, cb);
};

// 遍历 src/locale/lang 目录下的所有文件
fileList
	// 只处理 js 文件
  .filter(function(file) {
    return /\.js$/.test(file);
  })
  .forEach(function(file) {
  	// 获取文件的文件名
    var name = basename(file, '.js');

  	// 调用转换函数，利用 babel-core、add-module-exports插件、transform-es2015-modules-umd插件
  	// 处理 src/locale/lang 下的文件，生成 UMD 格式的文件，输出至 lib/umd/locale 目录下
    transform(file, name, function(err, result) {
      if (err) {
        console.error(err);
      } else {
        var code = result.code;

        code = code
          .replace('define(\'', 'define(\'element/locale/')
          .replace('global.', 'global.ELEMENT.lang = global.ELEMENT.lang || {}; \n    global.ELEMENT.lang.');
        save(resolve(__dirname, '../../lib/umd/locale', file)).write(code);

        console.log(file);
      }
    });
  });
```

#### `build/bin/gen-cssfile.js`

生成 `packages/theme-chalk/index.scss` 样式总入口文件。全量引入组件时，引用该样式如下：`import 'packages/theme-chalk/src/index.scss'`。

```javascript
var fs = require('fs');
var path = require('path');
// 获取项目组件清单
var Components = require('../../components.json');
// 主题列表
var themes = [
  'theme-chalk'
];
// 获取所有组件名称
Components = Object.keys(Components);
// 组件的 basepath，通过拼接可得到每个组件文件的路径
var basepath = path.resolve(__dirname, '../../packages/');

// 判断文件是否存在
function fileExists(filePath) {
  try {
    return fs.statSync(filePath).isFile();
  } catch (err) {
    return false;
  }
}

// 遍历主题
themes.forEach((theme) => {
  // 不是默认主题，使用 sass 预处理器，后缀为 .scss
  var isSCSS = theme !== 'theme-default';
  // 导入基础样式语句
  var indexContent = isSCSS ? '@import "./base.scss";\n' : '@import "./base.css";\n';
  // 遍历系统中的所有组件
  Components.forEach(function(key) {
    // 不引入 icon、option、option-group 的样式
    if (['icon', 'option', 'option-group'].indexOf(key) > -1) return;
    // 判断样式文件格式
    var fileName = key + (isSCSS ? '.scss' : '.css');
    // 导入组件样式语句
    indexContent += '@import "./' + fileName + '";\n';
    // 组件样式文件路径
    var filePath = path.resolve(basepath, theme, 'src', fileName);
    // 判断组件样式文件是否存在，不存在则创建遗漏组件样式文件
    if (!fileExists(filePath)) {
      fs.writeFileSync(filePath, '', 'utf8');
      console.log(theme, ' 创建遗漏的 ', fileName, ' 文件');
    }
  });
  // 生成 packages/theme-chalk/src/index.scss
  fs.writeFileSync(path.resolve(basepath, theme, 'src', isSCSS ? 'index.scss' : 'index.css'), indexContent);
});
```

#### `build/bin/gen-indices.js`

使用 `algoliasearch` 轻松实现文档全站搜索。

```javascript
const fs = require('fs');
const path = require('path');
// algoliasearch 服务
const algoliasearch = require('algoliasearch');
const slugify = require('transliteration').slugify;
// 获取 Admin API KEY 管理云端数据
const key = require('./algolia-key');

// 初始化 algoliasearch 服务，参数为 ApplicationID，Admin API KEY
const client = algoliasearch('4C63BTGP6S', key);
// 多语言对应的文档文件夹映射关系
const langs = {
  'zh-CN': 'element-zh',
  'en-US': 'element-en',
  'es': 'element-es',
  'fr-FR': 'element-fr'
};

// 遍历多语言的文档信息，把 examples/docs/{lang} 下的 .md 文件内容按照约定格式上传给 algoliasearch
['zh-CN', 'en-US', 'es', 'fr-FR'].forEach(lang => {
  // 获取当前语言的文件夹名称
  const indexName = langs[lang];
  // 初始化一个索引库
  const index = client.initIndex(indexName);
  // 清除索引
  index.clearIndex(err => {
    if (err) return;
		// 获取 examples/docs/{lang} 目录下文件列表
    fs.readdir(path.resolve(__dirname, `../../examples/docs/${ lang }`), (err, files) => {
      if (err) return;
      let indices = [];
      // 遍历 examples/docs/{lang} 目录下的 .md 文件
      files.forEach(file => {
        // 将文件名去除 .md 扩展名，得到组件名字
        const component = file.replace('.md', '');
        // 读取 examples/docs/{lang}/{file} 文件内容
        const content = fs.readFileSync(path.resolve(__dirname, `../../examples/docs/${ lang }/${ file }`), 'utf8');
        
        // 页面内容提取，移除自定义容器 ::: demo ``` 内容
        // 提取 h2 h3 标题和描述信息，输入格式如下
        // [
        // 	['## Button 按钮\r', '常用的操作按钮。\r\r'],
        // 	['### 基础用法\r', '\r基础的按钮用法\r\r\r\r'],
        // 	['### 禁用状态\r', '\r按钮不可用状态。\r\r\r\r']
        // ]
        //
        const matches = content
          .replace(/:::[\s\S]*?:::/g, '')
          .replace(/```[\s\S]*?```/g, '')
          .match(/#{2,4}[^#]*/g)
          .map(match => match.replace(/\n+/g, '\n').split('\n').filter(part => !!part))
          .map(match => {
            const length = match.length;
            if (length > 2) {
              const desc = match.slice(1, length).join('');
              return [match[0], desc];
            }
            return match;
          });
        
        // 将匹配内容格式化
        // {
        // 	component: 'button',
        // 	title: 'Button 按钮',
        // 	ranking: 2,
        // 	anchor: 'button-an-niu',
        // 	content: '常用的操作按钮。\r\r'
        // }
        indices = indices.concat(matches.map(match => {
          const isComponent = match[0].indexOf('###') < 0;
          // 去除 ## 或 ### 符号
          const title = match[0].replace(/#{2,4}/, '').trim();
          // 初始化 index 变量
          const index = { component, title };
          index.ranking = isComponent ? 2 : 1;
          index.anchor = slugify(title);
          index.content = (match[1] || title).replace(/<[^>]+>/g, '');
          return index;
        }));
      });

      index.addObjects(indices, (err, res) => {
        console.log(err, res);
      });
    });
  });
});
```

以 `button.md` 为例，提取后的页面索引内容如下：

```json	
[
  {
    "component": "button",
    "title": "Button 按钮",
    "ranking": 2,
    "anchor": "button-an-niu",
    "content": "常用的操作按钮。\r\r"
  },
  {
    "component": "button",
    "title": "基础用法",
    "ranking": 1,
    "anchor": "ji-chu-yong-fa",
    "content": "\r基础的按钮用法。\r\r\r\r"
  },
  {
    "component": "button",
    "title": "禁用状态",
    "ranking": 1,
    "anchor": "jin-yong-zhuang-tai",
    "content": "\r按钮不可用状态。\r\r\r\r"
  },
  ......
]
```

#### `build/bin/i18n.js`

用于项目官网的国际化。基于 `examples/i18n/page.json` 国际化配置和 `examples/pages/template` 目录下的所有模板文件，生成 `zh-CN`、`en-US`、`es`、`fr-FR` 等四种语言的 `.vue` 文件。

```javascript
var fs = require('fs');
var path = require('path');
// 页面国际化配置，支持 zh-CN、en-US、es、fr-FR 四种语言
var langConfig = require('../../examples/i18n/page.json');

// 遍历所有语言 zh-CN、en-US、es、fr-FR
langConfig.forEach(lang => {
  // 创建各语言页面目录，例如：examples/pages/zh-CN
  try {
    fs.statSync(path.resolve(__dirname, `../../examples/pages/${ lang.lang }`));
  } catch (e) {
    fs.mkdirSync(path.resolve(__dirname, `../../examples/pages/${ lang.lang }`));
  }

  // 遍历所有的页面，根据 {page}.tpl 生成对应语言的 .vue 文件
  Object.keys(lang.pages).forEach(page => {
    // 页面对应模板路径
    var templatePath = path.resolve(__dirname, `../../examples/pages/template/${ page }.tpl`);
    // 生成页面对应语言的 .vue 文件路径
    var outputPath = path.resolve(__dirname, `../../examples/pages/${ lang.lang }/${ page }.vue`);
    // 读取模板文件
    var content = fs.readFileSync(templatePath, 'utf8');
    // 读取页面的翻译信息列表 key/value
    var pairs = lang.pages[page];
		
    // 遍历翻译信息列表，模板变量内容替换
    Object.keys(pairs).forEach(key => {
      content = content.replace(new RegExp(`<%=\\s*${ key }\\s*>`, 'g'), pairs[key]);
    });

    // 生成 .vue 文件并写入内容
    fs.writeFileSync(outputPath, content);
  });
});
```

#### `build/bin/iconInit.js`

使用 `postcss` 解析 `icon.scss`，提取所有 `icon` 名字生成 `examples/icon.json`。

```javascript
// Postcss 是一个用 JavaScript 转换 CSS 的工具
var postcss = require('postcss');
var fs = require('fs');
var path = require('path');
// 读取 icon.scss 文件内容
var fontFile = fs.readFileSync(path.resolve(__dirname, '../../packages/theme-chalk/src/icon.scss'), 'utf8');

// postcss 解析 css 返回对象的 nodes 属性数组
// node 对象示例
// {
// 	raws: { before: '\n\n', between: ' ', semicolon: true, after: '\n' },
// 	type: 'rule',
// 	nodes: [ [Declaration] ],
// 	parent: Root {
// 		raws: [Object],
// 		type: 'root',
// 		nodes: [Circular *1],
// 		source: [Object]
// 	},
// 	source: { start: [Object], input: [Input], end: [Object] },
// 	selector: '.el-icon-chat-square:before'
// }
var nodes = postcss.parse(fontFile).nodes;

// 存放 icon 名称数组
var classList = [];

// 遍历 nodes
nodes.forEach((node) => {
  // node 获取选择器名称
  var selector = node.selector || '';
  var reg = new RegExp(/\.el-icon-([^:]+):before/);
  var arr = selector.match(reg);

  // 正则匹配提取 icon 名称
  if (arr && arr[1]) {
    classList.push(arr[1]);
  }
});

classList.reverse(); // 希望按 css 文件顺序倒序排列

// 将 icon 名称数组写入至 examples/icon.json
fs.writeFile(path.resolve(__dirname, '../../examples/icon.json'), JSON.stringify(classList), () => {});
```

`icon.json` 在 `examples/entry.js` 文件中导入，挂载到 `Vue.protyotype`。用于 Icon 图标文档页生成所有的图标集合，如下代码所示：

```javascript
import icon from './icon.json';
// Icon 列表页用
Vue.prototype.$icon = icon;
```

#### `build/bin/new-lang.js`

为网站添加新语言。如：`make new-lang fr`，添加新语言配置到 `component.json`、`page.json`、`route.json`、`nav.config.json` 等文件中，配置默认复制 `en-US` 语言设置，新建对应文件夹 `examples/docs/fr`。

```javascript
'use strict';
/**
 * 为网站添加新语言，例如：make new-lang fr
*/
console.log();
// 监听 exit 事件，在进程退出前进行操作
process.on('exit', () => {
  console.log();
});

// 判断第三个参数是否传入，执行添加语言
// argv 数组：[node, build/bin/new-lang.js, fr]
if (!process.argv[2]) {
  console.error('[language] is required!');
  process.exit(1);
}

var fs = require('fs');
const path = require('path');
const fileSave = require('file-save');
// 添加新的语言
const lang = process.argv[2];
// const configPath = path.resolve(__dirname, '../../examples/i18n', lang);

// 添加到 components.json
const componentFile = require('../../examples/i18n/component.json');
if (componentFile.some(item => item.lang === lang)) {
  console.error(`${lang} already exists.`);
  process.exit(1);
}
let componentNew = Object.assign({}, componentFile.filter(item => item.lang === 'en-US')[0], { lang });
componentFile.push(componentNew);
fileSave(path.join(__dirname, '../../examples/i18n/component.json'))
  .write(JSON.stringify(componentFile, null, '  '), 'utf8')
  .end('\n');

// 添加到 page.json
const pageFile = require('../../examples/i18n/page.json');
let pageNew = Object.assign({}, pageFile.filter(item => item.lang === 'en-US')[0], { lang });
pageFile.push(pageNew);
fileSave(path.join(__dirname, '../../examples/i18n/page.json'))
  .write(JSON.stringify(pageFile, null, '  '), 'utf8')
  .end('\n');

// 添加到 route.json
const routeFile = require('../../examples/i18n/route.json');
routeFile.push({ lang });
fileSave(path.join(__dirname, '../../examples/i18n/route.json'))
  .write(JSON.stringify(routeFile, null, '  '), 'utf8')
  .end('\n');

// 添加到 nav.config.json
const navFile = require('../../examples/nav.config.json');
navFile[lang] = navFile['en-US'];
fileSave(path.join(__dirname, '../../examples/nav.config.json'))
  .write(JSON.stringify(navFile, null, '  '), 'utf8')
  .end('\n');

// docs 下新建对应文件夹
try {
  fs.statSync(path.resolve(__dirname, `../../examples/docs/${ lang }`));
} catch (e) {
  fs.mkdirSync(path.resolve(__dirname, `../../examples/docs/${ lang }`));
}

console.log('DONE!');
```

#### `build/bin/new.js`

创建新组件 `package`，自动创建组件相关文件和初始组件的全局配置。例如 `make new button` 按钮，步骤如下：

- 创建的新组件添加到组件清单 `components.json` 中。
- 注意样式入口文件 `packages/theme-chalk/src/index.scss` 添加组件导入语句。
- 在 `types/element-ui.d.ts` 自动引入新组件类型声明。
- 创建 `package`
  - 创建组件文件 `packages/button/index.js` 和 `packages/button/src/main.vue`
  - 创建多语言组件文档 `examples/docs/{lang}/button.md`
  - 创建单元测试文件 `test/unit/specs/button.spec.js`
  - 创建组件样式文件 `packages/theme-chalk/src/button.scss`
  - 创建组件类型声明文件 `types/button.d.ts`
- 更新 `nav.config.json`，添加新组件导航信息 (组件菜单下左侧的二级导航)

```javascript
'use strict';

/**
 * 创建组件：make new <component-name> [中文名]
 * 例如：make new button 按钮
*/

console.log();
// 监听 exit 事件，在进程退出前进行操作
process.on('exit', () => {
  console.log();
});

// 判断第三个参数 component-name 是否传入
// argv 数组：[node, build/bin/new.js, button, 按钮]
if (!process.argv[2]) {
  console.error('[组件名]必填 - Please enter new component name');
  process.exit(1);
}

const path = require('path');
const fs = require('fs');
const fileSave = require('file-save');
const uppercamelcase = require('uppercamelcase');
// 新创建的组件名 component-name
const componentname = process.argv[2];
// 新创建的组件的中文名，若没有设置，默认使用 component-name
const chineseName = process.argv[3] || componentname;
// 大驼峰命名格式化
const ComponentName = uppercamelcase(componentname);
// 组件的 package 路径
const PackagePath = path.resolve(__dirname, '../../packages', componentname);
const Files = [
  // 创建组件入口 packages/button/index.js
  {
    filename: 'index.js',
    content: `import ${ComponentName} from './src/main';

/* istanbul ignore next */
${ComponentName}.install = function(Vue) {
  Vue.component(${ComponentName}.name, ${ComponentName});
};

export default ${ComponentName};`
  },
  // 创建组件 pacakges/button/src/main.vue
  {
    filename: 'src/main.vue',
    content: `<template>
  <div class="el-${componentname}"></div>
</template>

<script>
export default {
  name: 'El${ComponentName}'
};
</script>`
  },
  // 创建多语言组件文档，路径：examples/docs/{lang}/button.md
  // 若添加了多语言，需要在此处新增配置
  {
    filename: path.join('../../examples/docs/zh-CN', `${componentname}.md`),
    content: `## ${ComponentName} ${chineseName}`
  },
  {
    filename: path.join('../../examples/docs/en-US', `${componentname}.md`),
    content: `## ${ComponentName}`
  },
  {
    filename: path.join('../../examples/docs/es', `${componentname}.md`),
    content: `## ${ComponentName}`
  },
  {
    filename: path.join('../../examples/docs/fr-FR', `${componentname}.md`),
    content: `## ${ComponentName}`
  },
  // 创建单元测试文件，路径：test/unit/specs/button.spec.js
  {
    filename: path.join('../../test/unit/specs', `${componentname}.spec.js`),
    content: `import { createTest, destroyVM } from '../util';
import ${ComponentName} from 'packages/${componentname}';

describe('${ComponentName}', () => {
  let vm;
  afterEach(() => {
    destroyVM(vm);
  });

  it('create', () => {
    vm = createTest(${ComponentName}, true);
    expect(vm.$el).to.exist;
  });
});
`
  },
  // 创建组件样式文件，路径：packages/theme-chalk/src/button.scss
  {
    filename: path.join('../../packages/theme-chalk/src', `${componentname}.scss`),
    content: `@import "mixins/mixins";
@import "common/var";

@include b(${componentname}) {
}`
  },
  // 创建组件类型声明文件，路径：types/button.d.ts
  {
    filename: path.join('../../types', `${componentname}.d.ts`),
    content: `import { ElementUIComponent } from './component'

/** ${ComponentName} Component */
export declare class El${ComponentName} extends ElementUIComponent {
}`
  }
];

// 添加到 components.json
const componentsFile = require('../../components.json');
// 判断组件是否已经存在
if (componentsFile[componentname]) {
  console.error(`${componentname} 已存在.`);
  process.exit(1);
}
// 添加新组件到组件清单
componentsFile[componentname] = `./packages/${componentname}/index.js`;
fileSave(path.join(__dirname, '../../components.json'))
  .write(JSON.stringify(componentsFile, null, '  '), 'utf8')
  .end('\n');

// 添加到 index.scss
const sassPath = path.join(__dirname, '../../packages/theme-chalk/src/index.scss');
const sassImportText = `${fs.readFileSync(sassPath)}@import "./${componentname}.scss";`;
fileSave(sassPath)
  .write(sassImportText, 'utf8')
  .end('\n');

// 添加到 element-ui.d.ts
const elementTsPath = path.join(__dirname, '../../types/element-ui.d.ts');

let elementTsText = `${fs.readFileSync(elementTsPath)}
/** ${ComponentName} Component */
export class ${ComponentName} extends El${ComponentName} {}`;

const index = elementTsText.indexOf('export') - 1;
const importString = `import { El${ComponentName} } from './${componentname}'`;

elementTsText = elementTsText.slice(0, index) + importString + '\n' + elementTsText.slice(index);

fileSave(elementTsPath)
  .write(elementTsText, 'utf8')
  .end('\n');

// 创建 package
Files.forEach(file => {
  fileSave(path.join(PackagePath, file.filename))
    .write(file.content, 'utf8')
    .end('\n');
});

// 添加到 nav.config.json
const navConfigFile = require('../../examples/nav.config.json');

Object.keys(navConfigFile).forEach(lang => {
  // 获取组件分组导航信息，详细见组件菜单下左侧的二级导航
  let groups = navConfigFile[lang][4].groups;
  // 在 groups 中的 Others 分组中添加组件信息
  groups[groups.length - 1].list.push({
    path: `/${componentname}`,
    title: lang === 'zh-CN' && componentname !== chineseName
      ? `${ComponentName} ${chineseName}`
      : ComponentName
  });
});

// 更新 nav.config.json
fileSave(path.join(__dirname, '../../examples/nav.config.json'))
  .write(JSON.stringify(navConfigFile, null, '  '), 'utf8')
  .end('\n');

console.log('DONE!');
```

> 注意，如果项目中增加了新的语言，需要在 Files 数组中添加配置硬编码指定语言
>
> ```javascript
> const Files = [
>   {
>     filename: path.join("../../examples/docs/{new-lang}", `${componentname}.md`),
>     content: `## ${ComponentName}`
>   }
> ]
> ```

#### `build/bin/template.js`

用于监听 `examples/pages/template` 目录下模板文件是否改变，若存在改变会自动执行命令 `npm run i18n`，运行文件 `build/bin/i19n.js`，重新生成网站文件。

```javascript
const path = require('path');
// 模板路径
const templates = path.resolve(process.cwd(), './examples/pages/template');

// chokidar 一款用于文件监控的库
const chokidar = require('chokidar');
let watcher = chokidar.watch([templates]);

watcher.on('ready', function() {
  // 监听文件，文件夹的变化
  watcher
    .on('change', function() {
    // 若发生变化，执行命令 npm run i18n，重新生成文件
      exec('npm run i18n');
    });
});

// 调用 CMD 方法
function exec(cmd) {
  return require('child_process').execSync(cmd).toString().trim();
}
```

#### `build/bin/version.js`

生成 `examples/version.js` 记录项目库版本信息：

```javascript
var fs = require('fs');
var path = require('path');
// 获取版本信息，若没有传入参数 VERSION，则从 package.json 文件获取版本信息
var version = process.env.VERSION || require('../../package.json').version;
var content = { '1.4.13': '1.4', '2.0.11': '2.0', '2.1.0': '2.1', '2.2.2': '2.2', '2.3.9': '2.3', '2.4.11': '2.4', '2.5.4': '2.5', '2.6.3': '2.6', '2.7.2': '2.7', '2.8.2': '2.8', '2.9.2': '2.9', '2.10.1': '2.10', '2.11.1': '2.11', '2.12.0': '2.12', '2.13.2': '2.13', '2.14.1': '2.14' };
if (!content[version]) content[version] = '2.15';
// 生成项目库版本信息
fs.writeFileSync(path.resolve(__dirname, '../../examples/versions.json'), JSON.stringify(content));
```

当执行 `npm run pub` 发布组件库时，会执行脚本 `build/release.sh`，会手动输入发布版本信息 (`read -p "Releasing $VERSION - are you sure? (y/n)" -n 1 -r`)，然后执行命令 `VERSION=$VERSION npm run dist`。

整个执行顺序：

1. `npm run pub` => 

2. `sh build/release.sh` => 

3. `输入 $VERSION` => 

4. `VERSION=$VERSION npm run dist` => 

5. `npm run build:file` => 

6. `node build/bin/version.js`。

#### `build/md-loader`

使用自定义 `markdown-loader` 对文件进行处理，将 `组件文档.md` 编译成 HTML。