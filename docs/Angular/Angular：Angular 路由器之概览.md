---
title: Angular：Angular 路由器之概览
date: 2021/11/22 15:11:00
tag: [深入理解 Angular]
---

# Angular：Angular 路由器之概览

## A Tree of States

Angular 程序是一棵组件树，有一些组件，如根组件，在程序中的位置不变。然而，我们需要动态渲染组件，其中一种方式就是使用路由器。通过使用路由模块的 `router-outlet` 指令，可以根据当前 url 在程序中某个位置渲染一些组件。例如，一个简单的笔记程序，对于 Case 1 url 你需要渲染 `HomeComponent` ，对于另一个 Case 2 url，可能需要渲染 `NoteComponent` 展示笔记列表：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/1XoVjy.jpg)

路由内部会把这些可被路由的组件称为路由状态，路由器会把程序中可被路由的组件作为一棵路由状态树，上面的示例中，`home` 页是一个 `route state`，笔记列表也是一个 `route state`。

路由的核心功能是可以在程序内进行组件导航，并且需要路由器在页面的某个出口处渲染组件，url 得随渲染状态进行对应的修改。为此路由器需要把相关的 url 和加载的组件绑定到一起，它通过让开发者定义路由状态，根据对应 url 来渲染对应组件。

通过在程序内导入 `RouterModule` 并在 `forRoot` 方法内定义 `Route` 对象数组，来定义路由状态，例如，一个简单程序的路由数组定义如下：

```typescript
import { RouterModule, Route } from '@angular/router';

const ROUTES: Route[] = [
  { path: 'home', component: HomeComponent },
  { path: 'notes',
    children: [
      { path: '', component: NotesComponent },
      { path: ':id', component: NoteComponent }
    ]
  },
];

@NgModule({
  imports: [
    RouterModule.forRoot(ROUTES)
  ]
})
```

传入的路由对象数组会被构造成如下的一棵路由状态树

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/8ILgJg.jpg)

重要的是，任何时候，一些路由状态 (即组件的组合) 会根据当前的 url 展示在用户屏幕上，该组合就是所谓的 `ActiveRouter`。一个 `ActiveRouter` 仅仅是所有路由状态树的子树而已，比如 `/nodes` 会被表示成如下的 `ActiveRouter`：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/H5LIEi.jpg)

有关攞配置有两个有趣的点：1、`RouterModule` 有一个 `forChild` 方法，也可以传入 `Router` 对象数组，然而尽管 `forChild` 和 `forRoot` 方法都包含路由指令和配置，但是 `forRoot` 可以返回 `Router` 对象，由于 `Router` 服务会改变浏览器 `location` 对象，而 `location` 对象又是一个全局单例对象，所以 `Router` 服务对象也必须全局单例。这就是你必须在根模块中只使用一次 `forRoot` 方法的原因，特性模块中应当应用 `forChild` 方法。2、当匹配到路由路径时，路由状态 `component` 属性定义的最贱就会被渲染在 `router-outlet` 指令挂载的地方，即渲染组件的动态元素。被渲染的组件会作为 `router-outler` 元素的兄弟节点而不是子节点，`router-outlet` 也可以层层嵌套，形成父子路由关系。

程序内部导航时，路由器对象会捕捉导航的 url，并与路由状态树中某个路由状态进行匹配。比如上文提到的路由状态树配置：

```typescript
const ROUTES: Route[] = [
  { path: 'home', component: HomeComponent },
  { path: 'notes',
    children: [
      { path: '', component: NotesComponent },
      { path: ':id', component: NoteComponent }
    ]
  },
];
```

`localhost:4200/notes:15` 会被匹配并加载 `NoteComponent` 组件，通过使用其提供的对象如 `ActivedRouter`，`NoteComponent` 组件能够访问到参数 15 并显示正确的笔记。带有冒号前缀的路径值，如 `:id`，也是一个必选参数，会匹配几乎所有值 (本例中是 15)，但是 `localhost:4200/iamerror` 会匹配失败而报错。

任意时刻，当前 url 表示当前程序激活路由状态的序列化形式。路由状态的改变会触发 url 的改变，同时，url 的改变也会触发当前激活路由状态的改变，它们表示的是同一个东西。在本系列的下一篇文章，会看到路由器对象根据路径匹配路由的内部算法。现在我们只需要它使用的是最先匹配策略，内部实现采用第一层级搜索，匹配 url 的第一个路径。

Angular 的路由器如何把程序中所有路由构建为一棵路由状态树是第一步。第二步是导航，描述如何从一个路由状态导航到另一个路由状态。

## 路由器生命周期

与组件生命周期相同，路由器在每一次路由状态切换时也存在生命周期

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/RgljKT.jpg)

每一次导航周期内，路由器会触发一系列事件。路由器对象也提供了监听路由事件的 `observable` 对象，用来定义一些自定义的逻辑，如加载动画，或辅助调试路由。值得关注的事件有：

- `NavigationStart` 表示单行周期开始。

- `NavigationCancel` 表示取消本次导航。比如，路由守卫拒绝导航到特定的路由。

- `RoutesRecognized` 当 url 匹配到对应的路由。

- `NavigationEnd` 导航成功结束。

每一个路由事件都是 `RouterEvent` 的子类

```typescript
const ROUTES: Route[] = [
  { path: 'home', component: HomeComponent },
  { path: 'notes',
    children: [
      { path: '', component: NotesComponent },
      { path: ':id', component: NoteComponent }
    ]
  },
];
```

根据上面定义的路由状态，当 url 导航到 `http://localhost:4200/notes/42`，看看发生什么有趣的事情。总体上来说主要包括以下几步：

- 很合重定向会被优先处理，因为只有将最终稳定的 url 匹配到路由状态才有意义，而本示例中没有重定向，所以 `http://localhost:4200/notes/42` 就是最终稳定的。
- 路由器使用最先匹配策略来匹配 url 和路由状态。本示例中，会优先匹配 `path: 'notes'`，然后就是 `path: 'id'`，匹配的路由组件是 `NoteComponent`。
- 由于匹配到了路由状态，所以路由器会检查该路由状态是否存在阻止导航的路由守卫。比如，只有登录用户才能看到笔记列表，而本示例中，没有路由守卫。同时，也没有定义 `resolve` 预先获取数据，所以路由器会继续执行导。
- 路由器会激活该路由状态的路由组件。
- 路由器完成导航，然后它会等待下一次路由状态的改变，重复以上过程

可以通过在 `forRoot` 方法内开启 `enableTrace: true` 选项，这样可以在浏览器控制台里看到打印的事件

```typescript
RouterModule.forRoot([
  ROUTES,
  {
    enableTracing: true
  }
]),
```

同理，组件可以通过注入路由器对象来访问到路由事件流，并订阅该事件流

```typescript
constructor(private router: Router) {
  this.router.events.subscribe( (event: RouterEvent) => console.log(event))
}
```

## 懒加载特性模块

随着程序越来越大，很多功能会被封装在单独的特性模块中，比如，一个卖书的网站可能包括书籍，用户等模块。问题是程序首次加载时不需要展示所有数据，所以没有必要在主模块中包含所有模块。否则这会导致主模块文件膨胀，并导致程序加载时间过长。最好的解决方法就是当用户导航到某些模块再按需加载这些模块，而 Angular 路由器的懒加载功能可以做到这点。

懒加载模块的实例如下

```typescript
// from the Angular docs https://angular.io/guide/lazy-loading-ngmodules#routes-at-the-app-level
{
  path: 'customers',
  loadChildren: 'app/customers/customers.module#CustomersModule'
}
```

传给 `loadChildren` 属性的值类型是字符串，而不是组件对象引用。在导入模块时，注意避免引用任何懒加载模块，否则编译时依赖该模块，导致 Angular 不得不在主代码包把它编译进来，破坏了懒加载的目的。

路由器在导航周期的重定向和 url 匹配阶段内，会开始加载懒加载模块

```typescript
/**
 * Returns the `UrlTree` with the redirection applied.
 *
 * Lazy modules are loaded along the way.
 */
export function applyRedirects(
    moduleInjector: Injector, configLoader: RouterConfigLoader, urlSerializer: UrlSerializer,
    urlTree: UrlTree, config: Routes): Observable<UrlTree> {
  return new ApplyRedirects(moduleInjector, configLoader, urlSerializer, urlTree, config).apply();
}
```

正如源码文件 `config.ts` 所描述的

> The router will use registered NgModuleFactoryLoader to fetch an NgModule associated with 'team'.Then it will extract the set of routes defined in that NgModule, and will transparently add those routes to the main configuration.
>
> 译（路由器会使用注册的 **NgModuleFactoryLoader** 来加载与 'team' 相关的模块，并把该懒加载模块中定义的路由集合，添加到主配置里。）

