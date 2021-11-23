---
title: Angular：Angular 路由器之理解路由器导航循环
date: 2021/11/22 15:11:00
tag: [深入理解 Angular]
---

# Angular：Angular 路由器之理解路由器导航循环

本文结束时，你将会明白在导航过程中，路由器将会询问自身三个问题

- 对于给定的 url，路由应该导航到哪一组组件？
- 路由可以导航到响应的组件？
- 路由是否应该为响应的组件预取数据

在此过程中，我们将详细了解以下内容

- 从开始到结束的整个路由过程
- 在导航期间和之后，路由器如何构建和使用 `ActiveRouterSnapshot` 对象树
- 使用 `<router-outlet>` 指令渲染内容

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/27J53w.jpg)

## 导航

在 Angular 中，我们构建单页应用，这意味着每当 url 改变时，我们实际上并不会从服务器加载新的页面。相反，路由器在浏览器中提供基于位置的导航，这对于单页应用来说必不可少。它允许我们在不刷新页面的情况下改变用户看到的内容以及 url。

每当 url 改变时就会发生导航 (路由)。我们需要一种在应用视图之间导航的方法，而带有 `href` 的标准锚标记将不起作用，因为这将触发整个页面的重载。这就是 Angular 为我们 `[routerLink]` 指令的原因。当点击时，它告诉路由器更新 url 并使用 `<router-outlet>` 指令渲染内容，而无需重载页面。

```html
<!-- without a routerLink -->
<a href='localhost:4200/users'>Users</a>  <!-- not what we want! -->
<!-- with a routerLink -->
<a [routerLink]="['/users']">Users</a>  <!-- router will handle this -->
```

对于每次导航，当路由器在屏幕上渲染新组件之前，会执行一系列步骤。这就是路由器导航生命周期。

成功导航的输出是：由 `<router-outlet>` 渲染的新组建以及一个被用作可查询的导航记录的树形 `ActivedRouter` 数据结构。出于我们的目的，只需要理解开发者和路由器都用 `ActivedRoute` 获取导航的相关信息，例如查询参数和组件内容

### 示例应用

我们将使用一个非常简单的应用作为运行示例。这是路由器配置

```typescript
const ROUTES = [
  {
    path: 'users',
    component: UsersComponent,
    canActivate: [CanActivateGuard],
    resolve: {
      users: UserResolver
    }
  }
];
```

该应用由一路由 `/users` 组成，它检查查询参数以查看用户是否登录 (`login = 1`)，然后显示它从模拟 API 服务检索到的用户名列表。

应用的细节不重要，我们只需要一个示例来查看导航循环

### 导航循环和路由事件

查看导航循环的一个好办法就是订阅路由器服务的 `events` observable

```typescript
constructor(private router: Router) {
  this.router.events.subscribe( (event: RouterEvent) => console.log(event))
}
```

在开发期间，也可以在路由器配置中传入选项 `enableTracing: true` 以观察导航循环

```typescript
RouterModule.forRoot(ROUTES, {
  enableTracing: true
})
```

在开发者控制台里，我们可以看到在导航到 `/users` 路由期间发出的事件

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/PlUxTv.jpg)

这些事件对于研究或调试路由器非常有用，也很容易利用它们，在导航期间显示加载消息

```typescript
ngOnInit() {
  this.router.events.subscribe(evt => { 
    if (evt instanceof NavigationStart) {
      this.message = 'Loading...';
      this.displayMessage = true;
    }
    if (evt instanceof NavigationEnd) this.displayMessage = false;
  });
}
```

我们运行个指向 `/users` 的导航

### 导航开始 (Navigation Start)

`events: NatigationStart`

在我们的示例应用中，用户首先点击了以下链接

```html
<a [routerLink]="['/users']" [queryParams]="{'login': '1'}">Authorized Navigation</a>
```

导航到 `/users`，传入查询参数 `login=1`

每当路由器检测到对路由链接指令的点击时，就会启动导航循环。启动导航也有其他的方式，例如路由服务的 `navigate` 和 `navigateByUrl` 方法。

以前，Angular 应用中可能同时运行多个导航 (因此需要导航 id)，但是由于此更改，一次只能拥有一个导航。

### URL 匹配以及重定向

`events: RoutesRecognized`

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/Djui1L.jpg)

首先，路由器会对路由器配置数组 (我们的示例中的 `ROUTES`) 进行深度优先搜索，并尝试将 URL `/uses` 和众多路由配置项的 `path` 属性相匹配，同时在此过程中应用重定向。

在我们的例子中，无需担心重定向，`/users` 将匹配到 `ROUTES` 中的以下配置

`{ path: 'users', component: UsersComponent, ... }`

如果匹配到的路径需要一个懒加载的模块，那么它将在此时加载。

路由器发出 `RoutesRecognized` 事件，以表明它找到了与 URL 匹配的项，以及一个将要导航到的组件 (UsersComponent)。这回答了路由器的第一个问题，“路由应该导航到哪个组件”。但是没有那么快，路由器必须确保它被允许导航到这个新组件。

进入路由守卫。

### 路由守卫 (Route Guards)

`events: GuardsCheckStart, GuardsCheckEnd`

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/2P9TrH.jpg)

路由守卫是布尔函数，路由器使用它来确定是否可以执行导航操作。作为开发人员，我们使用守卫 (guards) 来控制导航是否可以发生。在我们示例应用中，通过在路由配置中指定 `canActive` 守卫来检查用户的登录状态。

```typescript
{ path: 'users', ..., canActivate: [CanActivateGuard] }
```

守卫函数如下：

```typescript
canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): boolean {
    return this.auth.isAuthorized(route.queryParams.login);
}
```

当查询参数 `login = 1` 时，`isAuthorized` 会返回 `true`

此守卫传递 `login` 查询参数到 `auth` 服务中 (本例中为 `auth.service.ts`)

如果对 `isAuthorzed(route.queryParams.login)` 的调用返回了 `true` ，那么守卫传参成功。否则，守卫传参失败，路由器会发出 `NavigationCancel` 事件，然后中止整个导航。

其他的守卫 (guards) 包括 `canLoad` (模块是否被懒加载)、`canActivateChild` 、`canDeactivate` (存在表单填写的场景下，为了防止已填写的表单信息丢失，阻止用户直接从当前页面导航离开)。

守卫类似于服务，它们被注册为提供者 (`providers`)，并且是 `Injectable` 的。每当 URL 发生变化时，路由器都会运行守卫程序。

`canActive` 守卫在为路由获取数据之前运行，因为没有理由为不应该被激活的路由获取数据。一旦通过了守卫，路由器就回答了第二个问题：“我应该执行这个导航吗？”。路由器现在可以使用解析器 (resolvers) 预获取数据了。

### 路由解析器 (Route Resolvers)

`events: ResolverStart, ResolverEnd`

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/JUFpaS.jpg)

路由解析器是我们可以在导航的过程中，在路由器渲染内容之前用来预取数据的函数。类似于守卫，我们在路由配置中使用 `resolve` 属性指定解析器

```typescript
{ path: 'users', ..., resolve: { users: UserResolver } }
```

```typescript
// user.resolver.ts
export class UserResolver implements Resolve<Observable<any>> {
  constructor(private userService: MockUserDataService) {}

  resolve(): Observable<any> {
    return this.userService.getUsers();
  }
}
```

一旦路由器将 URL 匹配到某个路径，并且所有的守卫都通过了，它将调用 `UserResolver` 类中定义的 `resolve` 方法来获取数据。路由器将结果存储在 `ActivatedRoute` 服务的 `data` 对象的键 ( key ) `users` 下。可以通过订阅 `ActivatedRoute` 服务的 `data` observable 来读取此信息。

```typescript
activatedRouteService.data.subscribe(data => data.users);
```

在 `UsersComponent` 里使用 `ActivatedRoute` 服务，从解析器里检索数据

```typescript
export class UsersComponent implements OnInit {
  public users = [];

  constructor(private route: ActivatedRoute) {}

  ngOnInit() {
    this.route.data.subscribe(data => this.users = data.users);
  }
}
```

解析器允许我们在路由期间预获取组件数据。这种技术可以通过预获取数据来避免想用户显示部分加载的模板。在内部，路由器有个 `runResolve` 方法，它将执行解析器，并将结果存储在 `ActivatedRoute` 快照 (`snapshot`) 上。

```typescript
// pre_activation.ts
future.data = {...future.data,
               ...inheritedParamsDataResolve(future, paramsInheritanceStrategy).resolve};
```

一旦路由器处理了所有解析器，下一步就是使用适当的路由器出口 (`outlets`) 开始渲染组件。

### 激活路由 (Activated Route)

`events: ActivationStart, ActivationEnd, ChildActivationStart, ChildActivationEnd`

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/fflxpt.jpg)

现在是时候激活这些组件，并使用 `<router-outlet>` 显示它们了。路由器可以从 `ActivatedRouteSnapshots` 树中提取它需要的关于组件的信息，`ActivatedRouteSnapshot` 树是它在导航循环的前几个步骤构建的。

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/kH6Vsx.jpg)

所有的魔法发生在路由器的 `activateWith` 函数里

```typescript
activateWith(activatedRoute: ActivatedRoute, resolver: ComponentFactoryResolver|null) {
  if (this.isActivated) {
    throw new Error('Cannot activate an already activated outlet');
  }
  this._activatedRoute = activatedRoute;
  const snapshot = activatedRoute._futureSnapshot;
  const component = <any>snapshot.routeConfig !.component;
  resolver = resolver || this.resolver;
  const factory = resolver.resolveComponentFactory(component);
  const childContexts = this.parentContexts.getOrCreateContext(this.name).children;
  const injector = new OutletInjector(activatedRoute, childContexts, this.location.injector);
  this.activated = this.location.createComponent(factory, this.location.length, injector);
  // Calling `markForCheck` to make sure we will run the change detection when the
  // `RouterOutlet` is inside a `ChangeDetectionStrategy.OnPush` component.
  this.changeDetector.markForCheck();
  this.activateEvents.emit(this.activated.instance);
}
```

代码要点：

- 在第 9 行，`ComponentFactoryResolver` 用于创建 `UsersComponent` 的实例。路由器从第 7 行 `ActivatedRouteSnapshot` 中提取此信息。
- 在 12 行，组件被真正创建。`location` 是指向 `<router-outlet>` 的 `ViewContainerRef`。
- 在创建并激活组件之后，将调用上文未列出的 `activateChildRoutes`，为用户处理嵌套的 `<router-outlet>`，即存在子路由的场景。

路由器将在屏幕上渲染组件。如果所渲染的组件中有任何嵌套的 `<router-outlet>` 元素，路由器也将遍历并渲染这些元素。

### 更新 URL

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/kfhueZ.jpg)

导航循环的最后一步是更新 URL 到 `/users`。

```typescript
// router_link.ts
private updateTargetUrlAndHref(): void {
  this.href = this.locationStrategy.prepareExternalUrl(this.router.serializeUrl(this.urlTree));
}
```

路由器现在已尊卑好侦听另一个 URL 变更，并重新开始导航循环。

