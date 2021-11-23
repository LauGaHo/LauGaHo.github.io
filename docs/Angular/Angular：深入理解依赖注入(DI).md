# Angular：深入理解依赖注入 (DI)

如果之前没有深入理解 Angular 依赖注入系统，那么或许你会认为 Angular 程序内的根注入器包含所有合并的服务提供商，每一个组件都有它自己的注入器，延迟加载模块有它自己的注入器。

但是，仅仅知道这些或许不够。

在 Angular 中，有两个注入器层次结构：

- Module Injector (模块注入器)：使用 @NgModule( ) 或 @Injectable( ) 注解在此层次结构中配置 Module Injector
- Element Injector (元素注入器)：在每一个 DOM 上隐式创建

## 注入器树 (Injector Tree)

大多数开发者都知道，Angular 会创建根注入器，根注入器内的服务都是单例的。但是，貌似还有其他注入器是它的父级。

作为一名开发者，我想知道 Angular 是怎么构建注入器树的，下图是注入器树的顶层部分：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/g7kRmn.jpg)

这不是整棵树，目前还没有任何组件，后面会继续画树的其他部分，先看一下根注入器 AppModule Injector，因为他最常用。

## 根注入器 (root AppModule Injector)

我们知道 Angular 程序注入器就是上图中的 AppModule Injector，上文说了，这个根注入器包含所有中间模块的服务提供商，也就是说 (不翻译) ：

> If we have a module with some providers and import this module directly in AppModule or in any other module, which has already been imported in AppModule, then those providers become application-wide providers.

ComponentFactoryResolver 也会添加到这个根注入器当中，因为它是用于动态创建定义在 entryComponents 属性中的组件。

为了实例化根注入器，Angular 使用 `AppModule` 工厂函数，这个函数的位置在 `module.ngfactory.js`。

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/lIz3Si.jpg)

我们可以看到这个工厂函数返回的是一个融合了所有 Provider 的一个 Module 定义。相信大多数开发者都知道。

## 平台注入器 (Platform Injector)

事实证明，`AppModule` 根注入器的上一级有一个父注入器叫做 `NgZoneInjector`，他也有一个父注入器，名字叫做 `PlatformInjector`。

平台注入器通常包括了一些内置的 Provider，但是我们也可以在创建平台的时候往平台注入器中添加我们自定义的 Provider：

```typescript
const platform = platformBrowserDynamic([ { 
  provide: SharedService, 
  deps:[] 
}]);
platform.bootstrapModule(AppModule);
platform.bootstrapModule(AppModule2);
```

我们传进 platform 中的这个额外的 Provider必须是 StaticProviders。如果不了解 StaticProvider 和 Provider 的区别，可以点击这里查看 [Answer](https://stackoverflow.com/questions/48594944/can-deps-also-be-used-with-useclass/48595518#48595518)

虽然现在比较清晰 Angular 是如何根注入器和其父注入器是如何解析依赖的。但是，对于组件级别的注入器还是有一点迷惑，所以我们开始探索。

## EntryComponent and RootData

当我谈论 ComponentFactoryResolver 的时候，我提到了 `entryComponent`。这种 components 通常出现在 `NgModule` 中的 `bootstrap` 或者 `entryComponents` 数组中，Angular 路由也会使用他们来创建动态组件。

Angular 会帮所有的 `entryComponents` 创建宿主工厂函数，这些宿主工厂函数就是其他视图的根视图，这就意味着：

> Every time we create dynamic component angular creates root view with root data, that contains references to elInjector and ngModule injector.
>
> 参考译文：每次我们创建动态 Angular 组件时，都会使用根数据 ( `RootData` ) 创建根视图 ( `RootView` )。

```typescript
class ComponentFactory_ extends ComponentFactory<any>{
  create(
      injector: Injector, projectableNodes?: any[][], rootSelectorOrNode?: string|any,
      ngModule?: NgModuleRef<any>): ComponentRef<any> {
    if (!ngModule) {
      throw new Error('ngModule should be provided');
    }
    const viewDef = resolveDefinition(this.viewDefFactory);
    const componentNodeIndex = viewDef.nodes[0].element!.componentProvider!.nodeIndex;
    // 使用根数据创建根视图
    const view = Services.createRootView(
        injector, projectableNodes || [], rootSelectorOrNode, viewDef, ngModule, EMPTY_CONTEXT);
    // view.nodes 的访问器
    const component = asProviderData(view, componentNodeIndex).instance;
    if (rootSelectorOrNode) {
      view.renderer.setAttribute(asElementData(view, 0).renderElement, 'ng-version', VERSION.full);
    }
    // 创建组件
    return new ComponentRef_(view, new ViewRef_(view), component);
  }
}
```

该根数据 ( `RootData` ) 包含对 `elInjector` 和 `ngModule` 注入器的引用：

```typescript
function createRootData(
    elInjector: Injector, ngModule: NgModuleRef<any>, rendererFactory: RendererFactory2,
    projectableNodes: any[][], rootSelectorOrNode: any): RootData {
  const sanitizer = ngModule.injector.get(Sanitizer);
  const errorHandler = ngModule.injector.get(ErrorHandler);
  const renderer = rendererFactory.createRenderer(null, null);
  return {
    ngModule,
    injector: elInjector, projectableNodes,
    selectorOrNode: rootSelectorOrNode, sanitizer, rendererFactory, renderer, errorHandler
  };
}
```

现在猜测当我们启动一个 Angular 应用。

当我们执行下面的代码，会发生什么事情

```typescript
platformBrowserDynamic().bootstrapModule(AppModule);
```

实际上，会发生许多事情，但是我们只对 Angular 如何创建 entry component 感兴趣。

当应用程序 `ApplicationRef` 启动 `bootstrap` 时，会创建 `entryComponent`

```typescript
const compRef = componentFactory.create(Injector.NULL, [], selectorOrNode, ngModule);
```

该过程会使用根数据 `RootData` 创建根视图 `RootView`，同时创建根元素注入器，在这里 `elInjector` 为 `Injector.NULL`。

这里就是注入器树被分成元素注入器树和模块注入器树这两个平行树的地方。

Angular 会有规律的创建下级注入器，每当 Angular 创建一个在 `@Component()` 中指定了 ` providers` 的组件实例时，它也会为该实例创建一个新的子注入器。类似的，当在运行期间加载一个新的 `NgModule` 时，Angular 也可以为它创建一个拥有自己的提供者的注入器。

子模块和组件注入器彼此独立，并且会为所提供的服务分别创建自己的实例。当 Angular 销毁 `NgModule` 或组件实例时，也会销毁这些注入器以及注入器中的那些服务实例。

## Element Injector vs Module Injector

上一段时间，当懒加载开始广泛地被应用的时候，一个奇怪的现象在 Github 上被报告。依赖注入系统导致延迟加载模块的两倍实例。因此，一个新的设计被引入，从此开始，我们有了两颗平行的树，一个是用于元素，一个是用于模块的。

主要的规则如下：

当我们在组件或者在指令当中当中请求依赖，会使用 `Merge Injector` 去遍历元素注入器树，然后如果找不到依赖，就会切换到模块注入器树来解析依赖。

> Please note I don’t use phrase “component injector” but rather “element injector”.

### 什么是 Merge Injector

你们曾经是否写过这种代码：

```typescript
@Directive({
  selector: '[someDir]'
}
export class SomeDirective {
 constructor(private injector: Injector) {}
}
```

所以在这里的 `injector` 就是一个 `Merge Injector` (相应的，我们同样也可以将 Merge Injector 在组件的构造函数中注入)。

`Merge Injector` 本身没有任何值，它只是视图和元素定义的组合，它有以下定义：

```typescript
class Injector_ implements Injector {
  constructor(private view: ViewData, private elDef: NodeDef|null) {}
  get(token: any, notFoundValue: any = Injector.THROW_IF_NOT_FOUND): any {
    const allowPrivateServices =
        this.elDef ? (this.elDef.flags & NodeFlags.ComponentView) !== 0 : false;
    return Services.resolveDep(
        this.view, this.elDef, allowPrivateServices,
        {flags: DepFlags.None, token, tokenKey: tokenKey(token)}, notFoundValue);
  }
}
```

正如我们所看到的，`Merge Injector` 只是视图和元素定义的组合。当 Angular 解决依赖关系时，这个注入器就像是元素注入器树和模块注入器树之间的桥梁。

`Merge Injector` 同样也可以解析一些内置的依赖，类似于 `ElementRef`，`ViewContainerRef`，`TemplateRef`，`ChangeDetectorRef` 这些，更有趣的是，他还可以返回 `Merge Injector`

**基本上，每个元素都有一个 `merge injector` 即使你没有提供任何的 `Token`**

### 什么是 Element Injector

在 Angular 中，视图是模板的表示形式，它包含不同类型的节点，其中便有元素节点，元素注入器位于此节点上。

默认情况下 `ElementInjector` 为空，除非在 `@Directive( )` 或 `@Component( )` 的 `providers` 属性中进行配置。

当 Angular 为嵌套的 [html](http://www.fly63.com/tag/html) 元素创建元素注入器时，要么从父元素注入器继承它，要么直接将父元素注入器分配给子节点定义。

如果子 [html](http://www.fly63.com/tag/html) 元素上的元素注入器具有提供者，则应该继承该注入器。否则，无需为子组件创建单独的注入器，并且如果需要，可以直接从父级的注入器中解决依赖项。

我们都知道，Angular 通过传递视图定义创建工厂函数来解析模板。视图包含很多不同类型的节点，比如说：`deiective`，`text`，`provider`，`query` 等等，其中 `Element Node` 就是代表着 `DOM` 结构，实际上，`Element Injector` 就是驻留在这个 `Element Node` 上。Angular 保留了这个节点上关于 `Provider` 的信息在这两个属性上：

```typescript
export interface ElementDef {
  ...
  /**
   * visible public providers for DI in the view,
   * as see from this element. This does not include private providers.
   */
  publicProviders: {[tokenKey: string]: NodeDef}|null;
  /**
   * same as visiblePublicProviders, but also includes private providers
   * that are located on this element.
   */
  allProviders: {[tokenKey: string]: NodeDef}|null;
}
```

`Element Injector` 解析依赖的过程：

```typescript
const providerDef =
  (allowPrivateServices ? elDef.element!.allProviders :
    elDef.element!.publicProviders)![tokenKey];
if (providerDef) {
  let providerData = asProviderData(searchView, providerDef.nodeIndex);
  if (!providerData) {
    providerData = { instance: _createProviderInstance(searchView, providerDef) };
    searchView.nodes[providerDef.nodeIndex] = providerData as any;
  }
  return providerData.instance;
}
```

这个注入器包含组件 / 指令的实例，还有就是所有在组件 / 指令当中登记了的 `Provider`。

这些 `Providers` 的填充是在视图的实例化阶段，主要是由 Angular 的编译器中的 [ProviderElementContext](https://github.com/angular/angular/blob/master/packages/compiler/src/provider_analyzer.ts#L47) 处理的，如果我们深入到这个类会发现许多有趣的事。

比如说，当使用 `@Host` 装饰器时会有一些 **[限制](https://github.com/angular/angular/blob/master/packages/compiler/src/provider_analyzer.ts#L290-L298)**，可以使用宿主元素的 **`viewProviders`** 属性来解决这些限制，可以查看 https://medium.com/@a.yurich.zuev/angular-nested-template-driven-form-4a3de2042475 。

另外的一个例子是，如果我们在组件上挂载指令，然后我们在组件和在指令上都拥有一个一样的 `token`，这个时候指令上的依赖会胜出。

### 依赖算法 (Resolution Algorithm)

要想明白依赖解析算法如何工作，我们需要对视图和视图父元素这个概念熟悉。

如果我们有一个根组件 `AppComponent`，他的模板为 `<child></child>`，然后我们有三个视图，如下：

```html
HostView_AppComponent
    <my-app></my-app>
View_AppComponent
    <child></child>
View_ChildComponent
    some content
```

依赖解析算法会根据视图层级进行展开：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/HIC19t.jpg)

> - 首先查看子元素注入器，进行检查 `elRef.element.allProvider|publicProviders`。
> - 然后遍历所有视图父元素 ①，并检查元素注入器中的提供者。
> - 如果下一个视图父元素等于 `null` ②，则返回 `startView` ③，检查 `startView.rootData.elInjector` ④。
> - 只有在找不到令牌的情况下，才检查 `startView.rootData.module.injector` ⑤。

如果我们在子组件中查找依赖，首先他会查看 `childComponent` 的 `Injector`，也就是查看 `elRef.element.allProviders|publicProviders`，然后会遍历所有的父视图查看 `Element Injector` 中的 `Provider`。直到视图父元素的值为 `null`。然后就会返回 `startView`，检查 `startView.rootData.eInjector` 如果最终 `token` 对应的依赖找不到，那么就会查看 `startView.rootData.module.injector`。

当向上遍历组件视图来解析依赖时，会搜索 **视图的父元素，而不是元素的父元素**。Angular 会使用 `viewParentEI( )` 函数获取视图父元素：

```typescript
/**
 * for component views, this is the host element.
 * for embedded views, this is the index of the parent node
 * that contains the view container.
 */
export function viewParentEl(view: ViewData): NodeDef|null {
  const parentView = view.parent;
  if (parentView) {
    return view.parentNodeDef !.parent;
  } else {
    return null;
  }
}
```

查找依赖详细代码如下：

```typescript
export function resolveDep(
    view: ViewData, elDef: NodeDef, allowPrivateServices: boolean, depDef: DepDef,
    notFoundValue: any = Injector.THROW_IF_NOT_FOUND): any {
  // 依赖注入被 @Value 装饰器修饰
  if (depDef.flags & DepFlags.Value) {
    return depDef.token;
  }
  const startView = view;
  // 依赖注入被 @Optional 修饰
  if (depDef.flags & DepFlags.Optional) {
    notFoundValue = null;
  }
  // 获取依赖的 token
  const tokenKey = depDef.tokenKey;

  // 依赖 token 为 ChangeDetectorRef
  if (tokenKey === ChangeDetectorRefTokenKey) {
    // directives on the same element as a component should be able to control the change detector
    // of that component as well.
    allowPrivateServices = !!(elDef && elDef.element !.componentView);
  }

  // 依赖被 @SkipSelf 修饰
  if (elDef && (depDef.flags & DepFlags.SkipSelf)) {
    allowPrivateServices = false;
    elDef = elDef.parent !;
  }

  // 将当前视图对象 view 赋值给 searchView
  let searchView: ViewData|null = view;
  // searchView 不为空就一直执行下去，除非提前返回
  while (searchView) {
    if (elDef) {
      switch (tokenKey) {
        // token 为 Renderer
        case RendererV1TokenKey: {
          const compView = findCompView(searchView, elDef, allowPrivateServices);
          return createRendererV1(compView);
        }
        // token 为 Renderer2
        case Renderer2TokenKey: {
          const compView = findCompView(searchView, elDef, allowPrivateServices);
          return compView.renderer;
        }
        // token 为 ElementRef  
        case ElementRefTokenKey:
          return new ElementRef(asElementData(searchView, elDef.nodeIndex).renderElement);
        // token 为 ViewContainerRef
        case ViewContainerRefTokenKey:
          return asElementData(searchView, elDef.nodeIndex).viewContainer;
        // token 为 TemplateRef  
        case TemplateRefTokenKey: {
          if (elDef.element !.template) {
            return asElementData(searchView, elDef.nodeIndex).template;
          }
          break;
        }
        // token 为 ChangeDetectorRef
        case ChangeDetectorRefTokenKey: {
          let cdView = findCompView(searchView, elDef, allowPrivateServices);
          return createChangeDetectorRef(cdView);
        }
        // token 为 Injector
        case InjectorRefTokenKey:
          return createInjector(searchView, elDef);
        // token 为其他类型
        default:
          const providerDef =
              (allowPrivateServices ? elDef.element !.allProviders :
                                      elDef.element !.publicProviders) ![tokenKey];
          if (providerDef) {
            let providerData = asProviderData(searchView, providerDef.nodeIndex);
            if (!providerData) {
              providerData = {instance: _createProviderInstance(searchView, providerDef)};
              searchView.nodes[providerDef.nodeIndex] = providerData as any;
            }
            return providerData.instance;
          }
      }
    }

    allowPrivateServices = isComponentView(searchView);
    elDef = viewParentEl(searchView) !;
    searchView = searchView.parent !;

    if (depDef.flags & DepFlags.Self) {
      searchView = null;
    }
  }

  const value = startView.root.injector.get(depDef.token, NOT_FOUND_CHECK_ONLY_ELEMENT_INJECTOR);

  if (value !== NOT_FOUND_CHECK_ONLY_ELEMENT_INJECTOR ||
      notFoundValue === NOT_FOUND_CHECK_ONLY_ELEMENT_INJECTOR) {
    // Return the value from the root element injector when
    // - it provides it
    //   (value !== NOT_FOUND_CHECK_ONLY_ELEMENT_INJECTOR)
    // - the module injector should not be checked
    //   (notFoundValue === NOT_FOUND_CHECK_ONLY_ELEMENT_INJECTOR)
    return value;
  }

  return startView.root.ngModule.injector.get(depDef.token, notFoundValue);
}
```

> - 读取依赖注入前的装饰器，例如：`@Host`、`@Optional`、`@Self`、`@SkipSelf`。并且根据这些装饰器，对传进来的 `viewData` 和 `elDef` 进行配置
> - 判断依赖注入 `token` 的类型，并且根据 `switch` 语句中的分类进行执行
> - 若是走到了 `default` 分支，那么将这样进行查找依赖：
>   - 先查看当前元素的元素注入器 (`elDef.allProviders|publicProviders`) 是否拥有给定 `token` 的依赖，若无执行下一步
>   - 将当前 `viewData.parent` 赋给新的 `viewData`，将当前 `viewData.parentDef.parent` 赋值给新的 `elDef`，然后进入下一个循环，也就是查看新的元素注入器 (`elDef.allProviders|publicProviders`) 中是否拥有给定依赖
> - 如果还是找不到对应的依赖，则直接去查找初始视图的根注入器 (`startView.root.injector.get(token)`)
> - 若还是找不到对应依赖，则查找 (`startView.root.ngModule.injector.get(token)`)

比如说，假设有如下一段小程序：

```typescript
@Component({
  selector: 'my-app',
  template: `<my-list></my-list>`
})
export class AppComponent {}

@Component({
  selector: 'my-list',
  template: `
    <div class="container">
      <grid-list>
        <grid-tile>1</grid-tile>
        <grid-tile>2</grid-tile>
        <grid-tile>3</grid-tile>
      </grid-list>
    </div>
  `
})
export class MyListComponent {}

@Component({
  selector: 'grid-list',
  template: `<ng-content></ng-content>`
})
export class GridListComponent {}

@Component({
  selector: 'grid-tile',
  template: `...`
})
export class GridTileComponent {
  constructor(private gridList: GridListComponent) {}
}
```

假设 `grid-tile` 组件依赖 `GridListComponent`，我们可以成功拿到该组件对象。但是这是怎么做到的？

**这里的视图父元素到底是什么？**

下面的步骤回答了这个问题：

1、查找 **起始元素**。我们的 `GridTileComponent` 拥有 `grid-tile` 这个元素选择器，因此我们需要查找匹配 `grid-tile` 选择器的元素

2、查找 **模板**。查找 `grid-tile` 元素所属的模板 (`MyListComponent template`)。

3、确定该元素的视图，如果没有父嵌入视图，则为组件视图，否则为嵌入视图。`grid-tile` 元素之上没有任何的 `ng-template` 或者 `*structuralDirective`，所以这里是组件视图 `View_MyListComponent`。

4、查找视图的父元素。这里是视图的父元素，而不是元素的父元素。

这里有两种情况：

- 对于嵌入视图，父元素则为包含该嵌入视图的视图容器，比如，假设 `grid-list` 上挂载有结构指令，则 `grid-tile` 视图的父元素则是 `div.container`。

```typescript
@Component({
  selector: 'my-list',
  template: `
    <div class="container">
      <grid-list *ngIf="1">
        <grid-tile>1</grid-tile>
        <grid-tile>2</grid-tile>
        <grid-tile>3</grid-tile>
      </grid-list>
    </div>
  `
})
export class MyListComponent {}
```

- 对于组件视图，父元素则为宿主元素

我们上面的小程序也就是组件视图，所以视图父元素是 `my-list` 元素，而不是 `grid-list`。

**现在，你可能想知道如果 Angular 跳过 `grid-list`，则它是怎么解析 `GridListComponent` 依赖的？**

关键在于 Angular 使用 **[原型链继承](https://github.com/angular/angular/blob/master/packages/core/src/view/view.ts#L87)** 来搜集 `Provider`：

> Each time we provide any token on an element, angular creates new `allProviders` and `publicProviders` array inherited from parent node, otherwise it just shares the same array with parent node.

这就表示了 `grid-tile` 包含当前视图内所有父元素的所有 `Provider`。

下图说明了 Angular 是如何为模板内元素收集 `Provider`：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/xnjleM.jpg)

正如我们所见，`grid-tile` 可以成功获取到 `GridListComponent` 从它自己的 Element Injector 中的 `allProviders` 属性，因为 `grid-tile` 的 Element Injector 包括了来自父元素的所有 Providers。

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/1_el4lEGLk_TSoZLUKlKK06Q.gif)

Element Injector 的 Provider 使用了原型链继承，导致了我们不能使用 multi 选项来提供同一个令牌多个服务，但是由于依赖注入系统很灵活，也有办法去解决这个可以，具体可以查看  [答案](https://stackoverflow.com/questions/49406615/is-there-a-way-how-to-use-angular-multi-providers-from-all-multiple-levels)

现在继续画注入树。

### Simple my-app -> child -> grand-child application

假设有如下简单程序：

```typescript
@Component({
  selector: 'my-app',
  template: `<child></child>`,
})
export class AppComponent {}

@Component({
  selector: 'child',
  template: `<grand-child></grand-child>`
})
export class ChildComponent {}

@Component({
  selector: 'grand-child',
  template: `grand-child`
})
export class GrandChildComponent {
  constructor(private service: Service) {}
}

@NgModule({
  imports: [BrowserModule],
  declarations: [
    AppComponent, 
    ChildComponent, 
    GrandChildComponent
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

我们有三层结构，并且 GrandChildComponent 依赖于 Service：

```typescript
my-app
   child
      grand-child(ask for Service dependency)
```

下图解释了 Angular 内部是如何解析 Service 依赖的：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/3nKSe9.jpg)

上面从 View_Child ① 的 `<grand-child>` 开始，并向上遍历查找所有视图的父元素，当视图没有父元素时，本实例中 `<my-app>` 没有父元素，则使用根视图的注入器进行查找 ②

```typescript
startView.root.injector.get(depDef.token, NOT_FOUND_CHECK_ONLY_ELEMENT_INJECTOR);
```

本实例中 `startView.root.injector` 就是 `NullInjector`，由于 `NullInjector` 没有任何 Provider，则 Angular 就会切换到模块注入器 ③：

```typescript
startView.root.ngModule.injector.get(depDef.token, notFoundValue);
```

所以 Angular 会按照以下顺序解析依赖：

```markdown
AppModule Injector 
        ||
        \/
    ZoneInjector 
        ||
        \/
  Platform Injector 
        ||
        \/
    NullInjector 
        ||
        \/
       Error
```

## 路由程序

让我们修改程序，添加路由器

```typescript
@Component({
  selector: 'my-app',
  template: `<router-outlet></router-outlet>`,
})
export class AppComponent {}
...
@NgModule({
  imports: [
    BrowserModule,
    RouterModule.forRoot([
      { path: 'child', component: ChildComponent },
      { path: '', redirectTo: '/child', pathMatch: 'full' }
    ])
  ],
  declarations: [
    AppComponent,
    ChildComponent,
    GrandChildComponent
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

这样视图树就类似为：

```markdown
my-app
   router-outlet
   child
      grand-child(dynamic creation)
```

现在让我们看看路由是如何创建动态组件：

```typescript
const injector = new OutletInjector(activatedRoute, childContexts, this.location.injector);                           
this.activated = this.location.createComponent(factory, this.location.length, injector);
```

这里 Angular 使用新的 rootData 对象创建一个新的根视图，同时传入 OutletInjector 作为根元素注入器 elInjector。OutletInjector 又依赖于父注入器 this.location.injector，该父注入器是 router-outlet 的元素注入器。

OutletInjector 是一种特别的注入器，行为有些向路由组件和父元素 router-outlet 之间的桥梁，该对象的代码可以看 **[这里](https://github.com/angular/angular/blob/master/packages/router/src/directives/router_outlet.ts#L144)**：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/Du0CuC.jpg)

## 延迟加载程序

最后，让我们把 GrandChildComponent 移到延迟加载模块，为此需要再子组件中添加 router-outlet，并修改路由配置：

```typescript
@Component({
  selector: 'child',
  template: `
    Child
    <router-outlet></router-outlet>
  `
})
export class ChildComponent {}
...
@NgModule({
  imports: [
    BrowserModule,
    RouterModule.forRoot([
      {
        path: 'child', component: ChildComponent,
        children: [
          { 
             path: 'grand-child', 
             loadChildren: './grand-child/grand-child.module#GrandChildModule'}
        ]
      },
      { path: '', redirectTo: '/child', pathMatch: 'full' }
    ])
  ],
  declarations: [
    AppComponent,
    ChildComponent
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

```typescript
my-app
   router-outlet
   child (dynamic creation)
       router-outlet
         +grand-child(lazy loading)
```

让我们为延迟加载程序画两棵独立的树：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/FE4nMJ.jpg)
