# Angular：使用 ViewContainerRef 技术操作 DOM

Angular 操作 DOM，总会发现会涉及到 `ElementRef`，`TemplateRef`，`ViewContainerRef` 和其他类。本文将详细讲解他们的使用思路。

Angular 可以运行在多种平台上，例如浏览器、移动端、`Web Worker` 等。所以，就需要在特定的平台的 API 和框架接口之间增加一个抽象层，用于屏蔽平台之间的差异性。Angular 中的这个抽象层包括如下引用类型：`ElementRef`，`TemplateRef`，`ViewRef`，`ComponentRef` 和 `ViewContainerRef`。本文将详细讲解这些引用类型以及他们如何用来操作 DOM。

## @ViewChild

在探索 DOM 抽象层之前，先了解下如何在组件 / 指令中访问它们。Angular 提供了一种叫 `DOM querys` 的技术，它们以 `@ViewChild` 和 `@ViewChildren` 装饰器的形式出现。两者功能类似，唯一区别是 `@ViewChild` 返回单个引用，`@ViewChildren` 返回由 `QueryList` 对象包装的好多个引用。本文将以 `ViewChild` 装饰器为例，后面描述时省略 `@` 符号。

通常这些装饰器与模板引用变量配合使用，模板引用变量可以理解为 DOM 元素的引用标识，类似 `html` 元素的 `id` 属性。你可以使用模板引用来标记一个 DOM 元素 (下文示例中的 `#tref`)，并在组件/指令仲使用 `ViewChild` 装饰器查询到它，比如

```typescript
@Component({
    selector: 'sample',
    template: `
        <span #tref>I am span</span>
    `
})
export class SampleComponent implements AfterViewInit {
    @ViewChild("tref", {read: ElementRef}) tref: ElementRef;

    ngAfterViewInit(): void {
        // outputs `I am span`
        console.log(this.tref.nativeElement.textContent);
    }
}
```

`ViewChild` 装饰器的基本语法是

```typescript
@ViewChild([reference from template], {read: [reference type]});
```

上例中你可以看到，我们把 `tref` 作为模板引用名称，并将 `ElementRef` 与该元素联系起来，第二个参数 `read` 是可选的，因为 Angular 会根据 DOM 元素的类型推断出该引用类型。例如，如果 `tref` 挂载的是类似于 `span` 的简单 html 元素，Angular 推断为 `ElementRef` 类型；如果它挂载的是 `template` 元素，Angular 推断为 `TemplateRef` 类型。一些引用类型如 `ViewContainerRef` 就不可以被 Angular 推断出来，所以必须在 `read` 参数中显式声明。其他如 `ViewRef` 不可以挂载在 DOM 元素中，所以必须手动在构造函数中编码构造出来。

## ElementRef

这时最基本的抽象类，如果查看它的类结构，会发现它仅仅包含与原生元素交互的方法，这对访问原生 DOM 元素很有用，比如

```typescript
// outputs `I am span`
console.log(this.tref.nativeElement.textContent);
```

然而，Angular 团队不鼓励这种写法，不仅因为这种方式会存在安全风险，而且还会让程序与渲染层紧耦合，这样就很难实现在多平台运行相同的应用程序。我认为这个问题并不是使用 `nativeElement` 导致的，而是使用特定的 DOM API 造成的，例如使用了 `textContent`。但是后文会看到，Angular 实现了操作 DOM 的整体思路模型，这样就不用必须调用平台指定的低层次抽象的 API，如 `textContent`。

使用 `ViewChild` 装饰器可以返回任何 DOM 元素对应的 `ElementRef`，但是由于所有组件挂载在自定义 DOM 元素中，所有指令也都应用在 DOM 元素上，所以组件和指令都可以通过 DI (Dependency Injection) 获取宿主元素的 `ElementRef` 对象，例如

```typescript
@Component({
    selector: 'sample',
    ...
export class SampleComponent{
	  constructor(private hostElement: ElementRef) {
          //outputs <sample>...</sample>
   		  console.log(this.hostElement.nativeElement.outerHTML);
	  }
	...
```

所有组件可以通过 DI 访问到它的宿主元素，但 `ViewChild` 装饰器经常被用来获取模板视图中的 DOM 元素。然而指令却相反，因为指令没有模板，所以主要用来获取挂载指令的宿主元素。

## TemplateRef

模板就是一组 DOM 元素的组合体，并且可以在整个应用中被重复使用。Angular 使用 `TemplateRef` 类来操作 `template` 元素 (`<ng-template>` 是 Angular 提供的类似于 `<template>` 原生 html 标签)

```typescript
@Component({
    selector: 'sample',
    template: `
        <ng-template #tpl>
            <span>I am span in template</span>
        </ng-template>
    `
})
export class SampleComponent implements AfterViewInit {
    @ViewChild("tpl") tpl: TemplateRef<any>;

    ngAfterViewInit() {
        let elementRef = this.tpl.elementRef;
        // outputs `template bindings={}`
        console.log(elementRef.nativeElement.textContent);
    }
}
```

Angular 框架从 DOM 中移除了 `template` 元素，并在其位置插入了注释，这时渲染后的样子

```html
<sample>
    <!--template bindings={}-->
</sample>
```

`TemplateRef` 是一个结构简单的抽象类，它的 `elementRef` 属性是对其宿主元素的引用，它还有一个 `createEmbeddedView` 方法。`createEmbeddedView` 方法非常有用，因为它可以创建一个视图并返回该视图的引用对象 `ViewRef`。

## ViewRef

该抽象类型表示一个 Angular 视图，在 Angular 世界里，视图是构建应用中 UI 的基础单位。它是可以同时创建和销毁的最小元素组合。Angular 鼓励开发者把 UI 作为一堆视图的组合，而不仅仅是 html 标签组成的树。

Angular 支持两种视图

- 内嵌视图 (EmbeddedView)，与 `TemplateRef` 关联
- 宿主视图 (HostView)，与 `Component` 关联

### 创建内嵌视图

模板仅仅是视图的蓝图，可以通过之前提到的 `createEmbeddedView` 方法来创建视图，比如

```typescript
ngAfterViewInit() {
    let view = this.tpl.createEmbeddedView(null);
}
```

### 创建宿主视图

宿主视图是在组件动态实例化时创建的，一个动态组件可以通过 `ComponentFactoryResolver` 创建

```typescript
constructor(private injector: Injector,
            private r: ComponentFactoryResolver) {
    let factory = this.r.resolveComponentFactory(ColorComponent);
    let componentRef = factory.create(injector);
    let view = componentRef.hostView;
}
```

在 Angular 中，每个组件都绑定着一个指定的注入器 (Injector) 实例，所以创建 `ColorComponent` 组件时传入当前组件 (即 `SampleComponent`) 的注入器。另外，别忘了，动态创建的组件，需要再 `ngModule` 中或者宿主组件中增加 EntryComponents 配置。

一旦视图创建，就可以使用 `ViewContainer` 插入 DOM 树中，下文主要探究这个功能。

## ViewContainerRef

视图容器可以挂载一个或多个视图

首先需要说的是，任何 DOM 元素都可以作为视图容器，然而有趣的是，对于绑定 `ViewContainer` 的 DOM 元素，Angular 不会把视图插入该元素的内部，而是追加到该元素的后边，这类似于 `router-outlet` 中插入组件的方式。

通常，比较好的方式是把 `ViewContainer` 绑定在 `ng-container` 元素上，因为 `ng-container` 元素会被渲染成注释，从而不会再 DOM 中引入多余的 html 元素。下面示例描述在组件模板中如何创建 `ViewContainer`

```typescript
@Component({
    selector: 'sample',
    template: `
        <span>I am first span</span>
        <ng-container #vc></ng-container>
        <span>I am last span</span>
    `
})
export class SampleComponent implements AfterViewInit {
    @ViewChild("vc", {read: ViewContainerRef}) vc: ViewContainerRef;

    ngAfterViewInit(): void {
        // outputs `template bindings={}`
        console.log(this.vc.element.nativeElement.textContent);
    }
}
```

如同其他 DOM 抽象类一样，`ViewContainer` 绑定到特殊的 DOM 元素，并可以通过 `element` 访问到。例如上例中，它绑定到 `ng-container` 元素上，并且渲染为 html 注释，所以输出会是 `template bindings={}`。

### 操作视图

`ViewContainer` 提供了一些操作视图 API

```typescript
class ViewContainerRef {
    ...m
    clear() : void
    insert(viewRef: ViewRef, index?: number) : ViewRef
    get(index: number) : ViewRef
    indexOf(viewRef: ViewRef) : number
    detach(index?: number) : ViewRef
    move(viewRef: ViewRef, currentIndex: number) : ViewRef
}
```

从上文我们已经知道内嵌视图和宿主视图的创建方式，当创建视图之后，就可以通过 `insert` 方法插入 DOM 中。下面示例描述如何通过 `ng-template` 创建内嵌视图，并在 `ng-container` 中插入该视图

```typescript
@Component({
    selector: 'sample',
    template: `
        <span>I am first span</span>
        <ng-container #vc></ng-container>
        <span>I am last span</span>
        <ng-template #tpl>
            <span>I am span in template</span>
        </ng-template>
    `
})
export class SampleComponent implements AfterViewInit {
    @ViewChild("vc", {read: ViewContainerRef}) vc: ViewContainerRef;
    @ViewChild("tpl") tpl: TemplateRef<any>;

    ngAfterViewInit() {
        let view = this.tpl.createEmbeddedView(null);
        this.vc.insert(view);
    }
}
```

通过上面的实现，最后的 `html` 看起来是

```html
<sample>
    <span>I am first span</span>
    <!--template bindings={}-->
    <span>I am span in template</span>

    <span>I am last span</span>
    <!--template bindings={}-->
</sample>
```

(从上文中知道是追加到 `ng-container` 后边，而不是插入到该 DOM 元素内部，因为在 Angular 中 `ng-container` 元素不会生成真实的 DOM 元素，所以在结构中不会发现这个追加的痕迹，如果把 `ng-container` 换成其他元素，则可以明显看到视图是追加在 `ViewContainer` 之后的：`<div _ngcontent-c4=""></div><span _ngcontent-c4>I am span in template</span>`)

此外，可以通过 `detach` 方法从 DOM 移除视图，其他的方法可以很容易通过方法名知道其含义，如通过 `index` 方法获得对应索引的视图引用，`move` 方法移动视图位置次序，或者使用 `remove` 方法从移除所有的视图。

### 创建视图

`ViewContainer` 也提供了手动创建视图 API

```typescript
class ViewContainerRef {
    element: ElementRef
    length: number

    createComponent(componentFactory...): ComponentRef<C>
    createEmbeddedView(templateRef...): EmbeddedViewRef<C>
    ...
}
```

上面两个方法是对上文我们手动操作的封装，可以传入模板引用对象或者组件工厂对象来创建视图，并将该视图出入到视图容器特定位置。

## ngTemplateOutlet 和 ngComponentOutlet

尽管知道 Angular 操作 DOM 的内部机制是好事，但是如果存在某种边界的方式就更好了。Angular 提供了两种跨界指令：`ngTemplateOutlet` 和 `ngComponentOutlet`。写作本文时，这两个指令都是实验性的，`ngComponentOutlet` 也将在 `4.0` 版本中可用。

### ngTemplateOutlet

该指令会把 DOM 元素标记为 `ViewContainer`，并插入由模板创建的内嵌视图，从而不需要在组件类中显式创建该内嵌视图。这意味着，上面实例中创建内嵌视图并插入 `#vc` DOM 元素的代码就可以重写为

```typescript
@Component({
    selector: 'sample',
    template: `
        <span>I am first span</span>
        <ng-container [ngTemplateOutlet]="tpl"></ng-container>
        <span>I am last span</span>
        <ng-template #tpl>
            <span>I am span in template</span>
        </ng-template>
    `
})
export class SampleComponent {}
```

从上面示例，我们不需要在组件类中写任何实例化视图的代码，非常方便。

### ngComponentOutlet

这个指令和 `ngTemplateOutlet` 很相似，区别是一个是用于宿主视图，另外一个是用于内嵌视图，使用如下

```html
<ng-container *ngComponentOutlet="ColorComponent"></ng-container>
```

## 总结

看似有很多新知识需要消化啊，但实际上 Angular 通过视图操作 DOM 的思路模型是很清晰和连贯的。你可以使用 `ViewChild` 查询模板引用变量来获得 Angular DOM 元素的引用对象；DOM 元素的最简单封装是 `ElementRef`；而对于模板，你可以使用 `TemplateRef` 来创建内嵌视图；而对于组件，可以使用 `ComponentRef` 来创建宿主视图，同时又可以使用 `ComponentFactoryResolver` 创建 `ComponentRef`。这两个创建的视图（即内嵌视图和宿主视图）又会被 `ViewContainerRef` 管理。最后，Angular 又提供了两个快捷指令自动化这个过程：`ngTemplateOutlet` 指令使用模板创建内嵌视图；`ngComponentOutlet` 使用动态组件创建宿主视图。