# Angular：对于 DOM 的新认知及优化技术

## 视图引擎 (View Engine)

假设需要从 `DOM` 中删除一个子元素，目前已经存在了父组件，父组件的模板中包含一个子组件 `A`，现在要做的是把 `A` 从父组件中移除

```typescript
@Component({
  ...
  template: `
    <button (click)="remove()">Remove child component</button>
    <a-comp></a-comp>
  `
})
export class AppComponent {}
```

解决上述需求，一个错误的方法是：使用 `Renderer` 或者 `native DOM API` 来直接删除 `<a-comp>` DOM 元素

```typescript
@Component({...})
export class AppComponent {
  ...
  remove() {
    this.renderer.removeChild(
       this.hostElement.nativeElement,      // parent App comp node
       this.childComps.first.nativeElement  // child A comp node
     );
  }
}
```

如果使用浏览器的审查元素功能，查看页面 `DOM` 节点情况，你会发现 `<a-comp>` 组件已经从 `DOM` 元素中移除了

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/1aE9sZ.jpg)

然而，如果查看控制台的输出，会发现打印结果中，仍然显示子节点的数量为 `1` 而不是 `0`。变更检测好像没有检测出子组件已经被移除了，下面是控制台输出的日志

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/03aiwb.jpg)

之所以出现这种状况，是因为 `Angular` 内部使用了一种数据结构来描述组件，通常称它为 `View` (视图) 或者 `ComponentView`。下图展示了视图和 `DOM` 之间的关联关系

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/0KiBJD.jpg)

每个视图由多个 (`View Node`) 构成，这些节点都包含有 `DOM` 元素的引用，当我们直接更改 `DOM` 元素时，视图节点并没有改变。下图演示了从 `DOM` 树中删除了组件 `<a-comp>`

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/dPt9Me.jpg)

所有的变更检测操作，包括 `ViewChildren` 都是运行在视图上，并非在 `DOM` 上。Angular 仍然会检测到 `<a-comp>`组件存在 (因为只是删除了 `DOM` 中的元素，视图中的引用仍然存在)，打印出子组件数量仍为 `1`。此外，变更检测也仍然会把 `<a-comp>` 作为检测对象。

> 上面的示例说明：不仅仅从 `DOM` 中删除子组件，应该避免删除任何由矿建创建的 HTML 元素，否则框架无法感知你删除了哪些元素。不过，你可以删除 Angular 框架无法感知到的元素，例如第三方库创建的元素。

## ViewContainer

`ViewContainer` 能够保证发生在其内部的 `DOM` 操作更安全 (保证 `View` 与 `DOM` 同步)，Angular 中所有内置的指令中都有用到它。它是一种特殊类型的 `View Node`，它既存在视图之中，也可以作为视图的容器，挂载其他视图

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/6nNFua.jpg)

如上图所示，它能接受两种类型的视图：内嵌视图 (EmbeddedView) 和宿主视图 (HostView)。

它们是 Angular 仅有的视图类型，它们之间的最大不同是用于创建它们的载体 (API 的输入不同) 不同。内嵌视图仅仅可以添加到 `ViewContainer` 中，而宿主视图除此之外，还可以添加到任意 `DOM` 元素中 (这个元素通常称为宿主元素)。

内嵌视图从模板创建，使用 `TemplateRef` 方法，宿主视图从视图创建，使用 `ComponentFactoryResolver` 方法。例如，用来启动 Angular 应用 (`AppComponent`) 的入口组件，在内部表示为附加到组件的宿主元素 (`<app-comp>`) 的宿主视图。

`ViewContainer` 提供了动态创建、操作、移除视图的 API，我们称它为动态视图，是相对框架中使用静态组件模板创建的视图。Angular 静态视图没有使用到 `ViewContainer`，它是在子组件中指定节点放一个子视图的引用。下图是静态视图的示例

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/XmrxIB.jpg)

从图中得知，视图中没有 `ViewContainer` 节点，子视图的引用直接关联到 `A` 组件。

## 动态操作视图

在开始创建和添加视图到 `ViewContainer` 之前，首先要把 `ViewContainer` 添加到组件的模板中，初始化它。任何元素都可以作为 `ViewContainer`，不过大部分情况，都使用 `<ng-container>` 元素，因为它在渲染时会被渲染成一个注释节点，因此 `DOM` 树中不会出现 `<ng-container>` 元素。

转换元素为 `ViewContainer`，可以在 `view query` 时使用 `{read: ViewContainer}`

```typescript
@Component({
 …
 template: `<ng-container #vc></ng-container>`
})
export class AppComponent implements AfterViewChecked {
  @ViewChild('vc', {read: ViewContainerRef}) viewContainer: ViewContainerRef;
}
```

获得 `ViewContainer` 的引用之后，那么就可以用它来创建动态视图了。

## 创建内嵌视图 (Embedded View)

创建内嵌视图，需要一个 `template`，在 Angular 中，我们使用 `<ng-template>` 元素来包装 `DOM` 元素，并且定义它为模板。并且可以通过 `view query` 的 `{read: TemplateRef}` 属性得到这个模板的引用

```typescript
@Component({
  ...
  template: `
    <ng-template #tpl>
        <!-- any HTML elements can go here -->
    </ng-template>
  `
})
export class AppComponent implements AfterViewChecked {
    @ViewChild('tpl', {read: TemplateRef}) tpl: TemplateRef<null>;
}
```

获取 `template` 引用之后，可以使用 `createEmbeddedView` 方法来创建和添加一个内嵌视图到 `ViewContainer` 之中。

```typescript
@Component({ ... })
export class AppComponent implements AfterViewInit {
    ...
    ngAfterViewInit() {
        this.viewContainer.createEmbeddedView(this.tpl);
    }
}
```

你可以在 `ngAfterViewInit` 生命周期钩子实现你的逻辑，因为 `view query` 在 `ngAfterViewInit` 周期时才初始化。此外，对于内嵌视图，你可以定义一个上下文对象 (`context object`)，来绑定 `<ng-template>` 中的值。可以参考 `createEmbeddedView()` 和 `NgTemplateOutlet` 来理解这个上下文对象。(`NgTemplateOutlet` 与 `createEmbeddedView` 功能上是相同的)。

## 创建宿主视图 (Host View)

创建宿主视图，需要使用 `component factory` 方法，更多关于动态创建组件的知识，可以查看这篇文章：[Here is what you need to know about dynamic components in Angular](https://indepth.dev/posts/1054/here-is-what-you-need-to-know-about-dynamic-components-in-angular)。

在 Angular 中，我们使用 `ComponentFactoryResolve service` 来获取 `component factory` 的引用

```typescript
@Component({ ... })
export class AppComponent implements AfterViewChecked {
  ...
  constructor(private r: ComponentFactoryResolver) {}
  ngAfterViewInit() {
    const factory = this.r.resolveComponentFactory(ComponentClass);
  }
}
```

## 移除一个视图

任何添加到 `view container` 的视图，都可以使用 `remove` 或者 `detach` 方法来移除它。这些方法在从 `view container` 中移除视图的同时，也会从 `DOM` 删除对应的元素。它们之间的区别是：`remove` 方法会销毁，此后这个视图无法再次被添加到 `view container` 中；`detach` 方法移除的视图，在后续还可以被重新添加到 `view container` 之中。这些特性，我们可以利用来优化 `DOM` 的操作。

所以正确的移除一个子组件或者 `DOM` 元素的方式，是需要首先创建嵌入视图或宿主视图，并把它们添加到 `view container` 中，并且使用 `view container` 中提供的 `remove` 或者 `detach` 等方法来移除它。

### 优化技术

在开发中，会遇到这样的场景：我们需要重复的隐藏和显示某个组件或模板元素，类似下图展示的那样

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/8.gif)

如果我们简单利用上文中学到的知识来解决这个问题，代码如下

```typescript
@Component({...})
export class AppComponent {
  show(type) {
    ...
    // a view is destroyed
    this.viewContainer.clear();
    
    // a view is created and attached to a view container      
    this.viewContainer.createComponent(factory);
  }
}
```

这样实现，会导致在切换隐藏和显示组件时，频繁地销毁和重建视图。

在这个特定的示例中，因为它是宿主视图，所以销毁和重新创建视图使用了 `component factory` 和 `createComponent` 方法。如果我们使用 `createEmbeddedView` 和 `TemplateRef`，那么就是内嵌视图被销毁和重建

```typescript
show(type) {
    ...
    // a view is destroyed
    this.viewContainer.clear();
    // a view is created and attached to a view container
    this.viewContainer.createEmbeddedView(this.tpl);
}
```

在理想状况下，我们仅需要创建一次视图，在后续的显示 / 隐藏过程中，可以复用这个视图。`ViewContainerRef` API 中提供了这样的方法，它支持添加视图到 `ViewContainer` 且在移除视图时不会测地销毁这个视图。

## ViewRef

`ComponentFactory` 和 `TemplateRef` 实现了 `view` 接口创建视图的方法，都可以用来创建一个视图。事实上，`ViewContainer` 在调用 `createComponent` 和 `createEmbeddedView` 方法时以及传递输入数据时，隐藏了一些细节。我们也可以自己利用这些方法来创建宿主视图或者内嵌视图，并且获得视图的引用，在 Angular 中这个视图的引用即是：`ViewRef` 以及它的子类。

## 创建一个宿主视图

下面是使用 `component factory` 来创建一个宿主视图，并且获得返回的引用的示例

```typescript
aComponentFactory = resolver.resolveComponentFactory(AComponent);
aComponentRef = aComponentFactory.create(this.injector);
view: ViewRef = aComponentRef.hostView;
```

在宿主视图下，`create` 方法返回的 `ComponentRef` 中包含组件关联的视图，可由 `hostView` 属性获得。

获得视图引用后，我们就可以使用 `insert` 方法来添加到 `ViewContainer` 中。结合上文提到的 `detach` 方法的特性，可以得出优化的方法为

```typescript
showView2() {
    ...
    // Existing view 1 is removed from a view container and the DOM
    this.viewContainer.detach();
    // Existing view 2 is attached to a view container and the DOM
    this.viewContainer.insert(view);
}
```

再提醒一下，这里使用的是 `detach` 方法，而不是 `clear` 或者 `remove` 方法，为的就是保证视图的引用没有被销毁，可以重复使用，不用再次创建。

## 创建一个内嵌视图

内嵌视图是利用 `ng-template` 进行创建的，创建的视图，由 `createEmbeddedView` 方法返回

```typescript
view1: ViewRef;
view2: ViewRef;
ngAfterViewInit() {
    this.view1 = this.t1.createEmbeddedView(null);
    this.view2 = this.t2.createEmbeddedView(null);
}
```

这里概念容易弄混，上文和下文提到的两种创建方式 内嵌 / 宿主视图的方式的区别就是：`create` 的行为是否由 `ViewContainer` 发起，实际使用效果没有差别。