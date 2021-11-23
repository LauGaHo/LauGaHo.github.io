# Angular：Angular 路由器之懒加载、AOT、及预加载

对于更快的首页加载来说，懒加载是一种有用的技术。通过懒加载，你的应用只需要将必要的启动代码发送到浏览器，就可以更快地加载。其他代码放在功能模块中，这些功能按需加载。

本文将深入了解路由器是如何实现懒加载的。

## Topics

- 懒路由配置是如何合并到根配置中的
- 懒加载如何与 AOT 与 JIT 编译一起工作的
- 路由器中的预加载是如何运作的

## Webpack, SystemJS 等类

当你使用像 Angular 这样的大型框架时，很容易忽略动态加载模块的真正含义。我们真正要做的是使用 Webpack 这样的工具将应用分割成不同的包 (Angular CLI 为我们处理这些包)，然后将这些包 pull 到我们应用中。

在本文中，我们不会重点讨论代码切割技术和模块加载器。

## 懒加载和路由器配置

要使用延迟加载，必须将应用分解为独立的 `NgModule` ，通常称为功能模块。

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/TvGoRH.jpg)

假设你已经将应用拆成功能模块，那么设置懒加载非常简单。`loadChildren` 属性用于路由器配置内部，指示模块应该懒加载。

```typescript
const ROUTES = [
 {
   path: 'lazy',
   loadChildren: './lazy-module/lazy.module#LazyModule',
 }
];
```

传给 `loadChildren` 的值是字符串。在 `#` 左边的所有字符串都是懒加载加载模块的路径，在 `#` 右侧的是 `NgModule` 的名字。

当用户导航到此路由时 (例如：`localhost: 4200/lazy`)，路由器将看到 `loadChildren` 属性，并开始加载功能模块 (在本例中为 `LazyModule`)。

> 必须小心避免在主包的任何位置添加任何类型的对功能模块的引用。否则，它将创建一个编译时依赖项，编译器将在主包中包含功能模块，而不是懒加载它。这就是为什么我们传递字符串作为 `loadChildren` 的值，而不是功能模块引用。

一旦功能模块加载完毕，它的路由器配置必须与应用主路由器配置 (在 `RouterModule.forRoot()` 中定义，通常位于 `app.module` 或者专用的 `app-routing.module` 中) 合并。

作为参考，我们 `LazyModule` 在路由中只有一条路由：

```typescript
// lazy.module.ts
const ROUTES = [
  { path: '', component: LazyComponent }
];

@NgModule({
  declarations: [
    LazyComponent
  ],
  imports: [
    RouterModule.forChild(ROUTES)
  ]
})
export class LazyModule { }
```

在导航的应用重定向阶段 (除非预加载，稍后会详细介绍)，使用 `forChild` 的功能模块中指定的配置被合并到 `_loadedConfig` 属性下的路由配置中。

```typescript
// apply_redirect.ts
return this.configLoader.load(ngModule.injector, route)
                  .pipe(map((cfg: LoadedRouterConfig) => {
                    route._loadedConfig = cfg;
                    return cfg;
                  }));
```

这可以通过检查路由器服务的 `config` 属性来验证

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/VfH8Nr.jpg)

这里的要点是，路由器必须加载任何懒模块，然后将它们的配置与应用的根路由器配置合并。一旦配置被合并，我们的应用程序就可以像往常一样访问懒模块中的路由。

### AOT 和懒加载

不管你是使用 Angular 的即时编译 (JIT) 还是提前编译 (AOT)，懒加载都能很好地运行。如果您正在使用 Angular CLI 构建应用，请运行

`ng serve --aot`

将使用 AOT 编译来构建和服务应用程序。你的所有功能模块都将提前编译，但它们仍然不会加载到应用中，直到它们被路由到。

你可以使用此命令

`ng build --aot=true`

然后你可以检查 `/dist` 文件夹来查看编译的文件。如果你使用的是我的示例库，你会看到：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/yUQXCT.jpg)

`lazy-module-lazy-module-ngfactory.js` 就是要懒加载到应用中的文件。它是一个预先编译成模块工厂的 `NgModule` 。这意味着我们的应用可以在加载后立即使用它，而不必在运行时编译成模块。

`ng serve` 和 `ng build` 默认使用 JIT 而不是 AOT。当我们不使用 AOT 构建时，`/dist` 会有模块而不是工厂

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/VlUdjD.jpg)

这意味着在懒加载期间，路由器必须将 `NgModule` 编译为工厂，然后才能使用它。

不管你使用 AOT 还是 JIT，在懒加载期间，都会影响 Angular 在运行时所采用的的一些步骤。要了解它是如何工作的，我们必须查看 `SystemJsNgModuleLoader` 的 `load` 方法

```typescript
// system_js_ng_module_factory_loader.ts
export class SystemJsNgModuleLoader implements NgModuleFactoryLoader {

  load(path: string): Promise<NgModuleFactory<any>> {
    const offlineMode = this._compiler instanceof Compiler;
    // offlineMode === true means AOT. The module is already compiled into a factory ahead of time
    return offlineMode ? this.loadFactory(path) : this.loadAndCompile(path);
  }

}
```

在使用 AOT 时，`offlineMode` 被设置为 `true` ，编译后的工厂被加载到应用程序中。否则 `NgModule` 将被下载并使用 `loadAndCompile` 进行编译，`loadAndCompile` 使用 Angular 的 `Compiler` 服务。

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/d3mqxt.jpg)

两者之间唯一的区别是，使用 JIT 时，懒模块必须在运行时编译为模块工厂。

不管你选择什么时候编译应用程序，懒加载都可以工作。和往常一样，AOT 会快一些，这通常是推荐的方法。

到目前为止，我们所看到的一切都发生在路由器的导航循环中。但是，可以在导航循环之外以编程方式加载模块。这个主题以及一些应用 (如动态路由) 将在以后的文章讨论。

### 预加载模块

我们已经看到懒加载模块时如何使初始包变小的。这意味着我们的应用将为用户更快地加载。但我们的工作还没完成。

假设我们有个大多数用户都可以访问的功能模块。一旦我们下载初始包并加载了应用，就没有理由在开始加载之前等待用户导航到那个流行的功能模块，最好是在后台加载它，这就是预加载发挥作用的地方。

预加载和懒加载一起工作。这时一种告诉 Angular 什么时候开始加载功能模块的方法。Angular 有两种默认的预加载策略：要么预加载所有内容，要么不预加载任何内容。

可以使用上面提到的两种默认预加载策略之一，也可以编写自己的自定义预加载策略。

你可以通过将一个选项传递到 `RouteModule.forRoot` ，来指定要使用哪一种预加载策略

```typescript
// preloading.ts
RouterModule.forRoot(ROUTES, {
  preloadingStrategy: CustomPreloadingStrategy | PreloadAllModules | NoPreloading
})
```

你可以指定自定义策略或者默认策略之一

自定义预加载策略被编写为实现 `PreloadingStrategy` 接口的服务。

### 路由器如何调度预加载

路由器需要某种提示来知道什么时候尝试预加载。在内部，路由器使用 `RouterPreloader` 的实例，该实例订阅路由器的 `events` observable 并侦听导航事件。每次发生`NavigationEnd` 事件时，预加载器都会检查是否有可以预加载的模块。

```text
// router_preloader.ts
setUpPreloading(): void {
  this.subscription =
      this.router.events
          .pipe(filter((e: Event) => e instanceof NavigationEnd), concatMap(() => this.preload()))
          .subscribe(() => {});
}
```

从路由器中侦听 `NavigationEnd` 事件，然后调用 `this.preload` 来递归路由配置。

有趣的事实: 你也可以在应用的任意位置订阅 `this.router.events`以查看所有的路由器事件。

如果你正在使用自定义策略，那么何时以及如何预加载模块的确切机制取决于自定义策略如何实现 `PreloadingStrategy` 的抽象方法 `preload`。然而，路由器总是在看到 `NavigationEnd` 时运行检查，看看是否可以预加载任何东西。

### 理解 CanLoad 守卫

最后但并非最不重要的，值得注意的是，懒加载有自己的路由器守卫程序，称为 **CanLoad**。它决定模块是否可以懒加载。如果返回`false`，模块甚至不会加载到浏览器中。

**注意，这个守卫程序会阻止所有预加载。两者不能同时使用**。

如果不是这样，我们可能会遇到用户在页面上的问题，并且在后台尝试预加载模块。这将激活 canLoad 守卫检查，该检查可能会失败，并将用户重定向到另一个页面(例如登录页面)，这绝对不是我们希望用户看到的场景。