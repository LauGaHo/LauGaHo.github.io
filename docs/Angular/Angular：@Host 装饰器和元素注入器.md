---
title: Angular：@Host 装饰器和元素注入器
date: 2021/11/22 15:11:00
tag: [深入理解 Angular]
---

# Angular：@Host 装饰器和元素注入器

Angular 依赖注入机制包含 `@Optional` 和 `@Self` 等影响依赖解析过程的装饰器，尽管它们字面意思就直接解释了作用，但是 `@Host` 却困扰了我好久，其源码对于该装饰器的描述

> Specifies that an injector should retrieve a dependency from any injector until reaching the host element of the current component.

由于网上大多教程都提到 Angular 的模块注入器和组件注入器，所以我认为 `@Host` 应该和多级注入器有关。`@Host` 可以用在子组件内，来限制只能在它自身和其父组件注入器解析依赖，所以我做了一个小示例来验证这个假设

```typescript
@Component({
    selector: 'my-app',
    template: `<a-comp></a-comp>`,
    providers: [MyAppService]
})
export class AppComponent {}

@Component({selector: 'a-comp', ...})
export class AComponent {
    constructor(@Host() s: MyAppService) {}
}
```

这里报错 No provider for MyAppService，有意思的是，如果我移除了 `@Host` 装饰器，`MyAppService` 就可以顺利从父组件注入器内解析出来，下面一探究竟。

关键就在于上文 `@Host` 装饰器描述的 'until' 一词上：

> …retrieve a dependency from any injector **until** reaching the host element

它意思是 `@Host` 装饰器会让依赖解析过程限制在当前组件模板，甚至都不包括其宿主元素 (注：在宿主元素 `a-comp` 上绑定含有 `MyService` 服务的指令 `ADirective`，是可以在 `AComponent` 的构造函数中解析出被 `@Host` 装饰的 `MyAppService` 服务)。这就是上面示例报错的原因 — Angular 不会从其宿主父组件注入器中解析依赖。

所以现在我们知道 `@Host` 装饰器不可以用来在子组件中解析来自父组件的依赖提供者，意味着该装饰器的依赖解析机制不可以用于多级组件注入器

> 所以应该使用什么样的多级注入器

实际上，除了模块注入器和组件注入器，Angular 还提供了第三种注入器，即多级元素注入器，它是由 HTML 元素和指令共同创建的。

## 元素注入器

Angular 会按照三个阶段来解析依赖，起始阶段就是使用多级元素注入器，然后当时多级组件注入器，最后是多级模块注入器。

对组件和模块注入器解析依赖的后两个阶段，应该很熟悉了，当延迟加载模块的时候，Angular 会创建多级模块注入器，多级组件注入器是模板中嵌套组件创建的，就内部实现而言，组件注入器也可以称为视图注入器，稍后会解释原因。

另外，多级元素注入器是 Angular 依赖注入系统中很少知道的功能，因为文档中没有写，但是这种注入器在依赖注入的起始阶段就使用了，这些多级元素注入器被用来解析由 `@Host` 装饰的依赖，所以让我们仔细研究一下这种注入器。

### 一个元素注入器

Angular 内部使用一种叫视图或组件视图的数据结构来表示组件 (注：可查看源码中 `ViewDefinition` 接口)。实际上，这就是把组件注入器称为视图注入器的由来，视图对象主要用来表示模板中有 HTML 元素创建的 DOM 节点的集合 (注：`@angular/compiler` 会编译你写的组件，生成结果由 `ViewDefinition` 接口来表示，不需要知道编译过程，只需要知道你写的带有 `@Component` 装饰的类不编译为新的类是无法直接运行的，且 HTML 模板就存在于薪类的属性中，即 Compiler HTML + Class into New Class，正因为 `@angular/compiler` 编译功能强大，所以可以在 HTML 模板里写很多不符合 HTML 语法的 HTML 代码，比如绑定指令等等)。所以每一个视图对象内部是有不同种类的视图节点组成的 (注：`ViewDefinition` 表示编译后的视图对象，视图又是由节点组成的，`@angular/core` 使用 `NodeDef` 接口表示所有节点，而节点又分很多种，使用 `NodeFlags` 来表示，其中最常见的 `TypeElement` 类型就是来标识 HTML 元素，使用 `ElementDef` 接口来表示)，最常见的视图节点类型是元素节点，用来表示对应的 `DOM` 元素，下图表示视图和 `DOM` 两者之间的关系

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/68WH7Y.jpg)

每个视图节点都是由节点定义对象实例化后创建的，节点定义对象包含描述节点的元数据，比如，像 `element` 类型的节点通常用来表示 DOM 元素，而这些元数据是有 `@angular/compiler` 的编译器编译组件模板和附着在模板上的指令生成的，下图表示视图节点定义与其对象的之间的关系

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/QprKnL.jpg)

元素节点定义描述了一个有趣的功能

> In Angular, a node definition that describes an HTML element defines its own injector. In other words, an HTML element in a component’s template defines its own element injector. And this injector can populated with providers by applying one or more directives on the corresponding HTML element.

直接看示例

假设组件模板中有个 `div` 元素，并且有两个指令挂载到上面

```typescript
@Component({
    selector: 'my-app',
    template: `<div a b></div>`
})
export class AppComponent {}

@Directive({ selector: '[a]' })
export class ADirective {}

@Directive({ selector: '[b]' })
export class BDirective {}
```

`@angular/compiler` 的编译器会编译模板生成视图，该视图中 DOM 元素 `div` 对应的元素节点定义对象包含如下元数据 (注：可查看 `ElementDef` 接口中的 `name` 和 `publicProvider` 属性)

```typescript
const DivElementNodeDefinition = {
    element: {
        name: 'div',
        publicProviders: {
            ADirective: referenceToADirectiveProviderDefinition,
            BDirective: referenceToBDirectiveProviderDefinition
        }
    }
}
```

正如你所见，节点对象定义了 `element.publicProviders` 属性，包含两个服务提供者 `ADirective` 和 `BDirective`，该属性作用类似于一个注入器，而 `referenceToADirectiveProviderDefinition` 和 `referenceToBDirectiveProviderDefinition` 就是附着在 `div` 元素上两个指令的实例。由于它们是由同一个元素注入器解析，所以你可以把其中一个指令注入到另外一个指令当中。当然，你不能在两个指令中相互注入依赖，因为这会导致依赖死锁。

所以下图说明了我们现在拥有的东西

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/dMcyU9.jpg)

注意宿主元素 `my-app` 存在于 `AppComponentView` 之外，因为它是属于父组件视图内的。

现在如果 `ADirective` 也包含服务提供者会发生什么？

```typescript
@Directive({
    selector: '[a]',
    providers: [ADirService]
})
export class ADirective {}
```

正如你所料，这个服务会被包含进由 `div` 创建的元素注入器中

```typescript
const divElementNodeDefinition = {
    element: {
        name: 'div',
        publicProviders: {
            ADirService: referenceToADirServiceProviderDefinition
            ADirective: referenceToADirectiveProviderDefinition,
            ADirective: referenceToADirectiveProviderDefinition
        }
    }
}
```

再放一次图，下图是现在的结果

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/MCJ7Kz.jpg)

### 多级元素注入器

上文我们只有一个 HTML 元素，嵌套 HTML 元素组成了 DOM 元素层级，组件视图内这些 DOM 元素组成了 Angular 依赖注入系统内多级元素注入器。

直接看示例

假设组件模板中有父子元素 `div`，同时还有两个指令 `A` 和 `B`。`A` 指令附着在父元素 `div` 上并提供 `ADirService` 服务，`B` 指令附着在子元素 `div` 上，但不提供任何服务。

代码如下：

```typescript
@Component({
    selector: 'my-app',
    template: `
        <div a>
            <div b></div>
        </div>
    `
})
export class AppComponent {}
@Directive({
    selector: '[a]',
    providers: [ADirService]
})
export class ADirective {}
@Directive({ selector: '[b]' })
export class BDirective {}
```

如果我们去探究 `@angular/compiler` 编译模板创建的元素节点定义对象，会发现存在两个 `element` 类型节点来描述 `div` 元素

```typescript
const viewDefinitionNodes = [
    {
        // element definition for the parent div
        element: {
            name: `div`,
            publicProviders: {
                ADirective: referenceToADirectiveProviderDefinition,
                ADirService: referenceToADirServiceProviderDefinition,
            }
        }
    },
    {
        // element definition for the child div
        element: {
            name: `div`,
            publicProviders: {
                BDirective: referenceToBDirectiveProviderDefinition
            }
        }
    }
]
```

正如上文发现的，每一个 `div` 元素定义都有一个 `publicProviders` 作为依赖注入容器。由于附着在父 `div` 元素还提供了 `ADirService` 服务，所以该服务也被加到父元素 `div` 的元素注入器内。

> 嵌套 HTML 结构创建了多级元素注入器

`ADirective` 指令提供一个服务，创建了两个层级元素注入器 — 上层级父注入器创建在 `div` 元素上，下层级子注入器创建在 `AppComponent` 上，这并不奇怪，因为组件也仅仅是带有指令的 HTML 元素。

### 创建元素注入器

当 Angular 为嵌套 HTML 元素创建注入器时，该注入器要么会继承父注入器，要么直接把父注入器复制给子注入器。如果子元素上挂载指令且该指令提供依赖服务，则子注入器会继承父注入器，也就是说，有挂载指令并提供服务的子元素创建的元素注入器，是会继承父注入器。另外，没有必要为子组件单独创建一个注入器，如有必要，可以直接使用父注入器来解析依赖。

下图说明了这个过程

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/Sizo1r.jpg)

#### 依赖解析过程

安装组件视图内的多级元素注入器，会大大简化依赖解析过程。Angular 使用 JavaScript 的原型链的属性查询机制来解析依赖，而不是一层层去查找父注入器去解析依赖

```typescript
elDef.element.publicProviders[tokenKey]
```

由于 JavaScript 的工作方式，访问 `publicProviders` 对象 `key` 对应的值，会直接从父元素注入器或原型链中解析出来。

## @Host 装饰器

我们之所以讨论元素注入器而不是 `@Host` 装饰器，是因为 `@Host` 会把元素注入器依赖解析过程限制在当前组件视图中。在一般依赖解析过程中，如果组件视图内的元素注入器不能解析一个令牌，Angular 依赖解析系统会遍历父视图，去使用组件/视图注入器来解析令牌，如果还没找到依赖，则遍历模块注入器去查找依赖。但一旦使用了 `@Host` 装饰器，整个依赖解析过程就会在第一阶段完成后停止解析，也就是说，元素注入器只在组件视图内解析依赖，然后就停止解析工作。

#### 示例

`@Host` 在表单指令中大量使用，比如，往 ngModel 指令里注入一个表单容器，并把由该指令实例化的表单控件对象 (FormControl) 注入到表单对象内，典型的模板驱动表单代码如下

```html
<form>
    <input ngModel>
</form>
```

实际上，NgForm 指令的选择器与 form DOM 元素匹配，该指令包含一个服务提供者，把自身注册为 ControlContainer 令牌指向的服务

```typescript
@Directive({
    selector: 'form',
    providers: [
        {
            provide: ControlContainer,
            useExisting: NgForm
        }
    ]
})
export class NgForm {}
```

