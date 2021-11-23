---
title: Angular：深入理解 Angular Modules
date: 2021/11/22 15:11:00
tag: [深入理解 Angular]
---

# Angular：深入理解 Angular Modules

Angular Modules 是一个相当复杂的话题，本文将深度解析 `Modules` 内部工作原理，争取做到消除一些常见的误会

## 模块封装

Angular 引入了模块封装的概念，这个和 ES 模块概念很相似，基本意思是所有声明类型，包括组件、指令和管道，只可以在当前模块内部，被其他声明的组件使用。比如，如果我在 App 组件中使用了 A 模块的 `acomp` 组件：

```typescript
@Component({
  selector: 'my-app',
  template: `
      <h1>Hello {{name}}</h1>
      <a-comp></a-comp>
  `
})
export class AppComponent { }
```

Angular 编译器就会抛出错误

>  Template parse errors: 'a-comp' is not a known element

这是因为 App 模块没有声明 `a-comp` 组件，如果我想要使用这个组件，就不得不引入 A 模块，就像这样

```typescript
@NgModule({
  imports: [..., AModule]
})
export class AppModule { }
```

上面描述的就是 **模块封装**。不仅如此，如果想要 `a-comp` 组件正常工作，得设置它为可以公开访问，即在 A 模块的 `export` 属性中导出这个组件

```typescript
@NgModule({
  ...
  declarations: [AComponent],
  exports: [AComponent]
})
export class AModule { }
```

同理，对于指令和管道，也必须遵循 **模块封装** 的规则

```typescript
@NgModule({
  ...
  declarations: [
    PublicPipe, 
    PrivatePipe, 
    PublicDirective, 
    PrivateDirective
  ],
  exports: [PublicPipe, PublicDirective]
})
export class AModule {}
```

需要注意的是，**模块封装** 原则不适用于在 `entryComponents` 属性中注册的组件，如果你在使用动态视图时，即实例化动态组件的方式引入 `a-comp` ，就不需要在 A 模块的 `export` 属性中导出 `a-comp` 组件，当然，还是得导入 A 模块。

大多数初学者都认为 `providers` 也有封装规则，但实际上没有。在 **非懒加载模块** 中声明的任何 `providers` 都可以在程序内的任何地方被访问。下文会详细解释原因。

## 模块层级

初学者最大的一个误解就是，一个模块导入其他模块之后会形成一个模块层级，认为该模块会成为这个被导入模块的父模块，从而形成一个类似模块树的层级，当然这么想也很合理。但实际上，不存在这样的模块层级。因为 **所有模块在编译阶段都会被合并**，所以导入和被导入模块之间不存在任何层级关系。

就像组件一样，Angular 编译器也会为根模块生成一个模块工厂，根模块就是你在 `main.ts` 中，以参数传入 `bootstrapModule()` 方法的模块

```typescript
platformBrowserDynamic().bootstrapModule(AppModule);
```

Angular 编译器使用 `createNgModuleFactory` 方法来创建该模块工厂，该方法需要几个参数

- Module Class Reference
- Bootstrap Components
- Component Factory Resolver With EntryComponents
- Definition Factory With Merged Module Providers

最后两点解释了为何 `providers` 和 `entryComponents` 没有模块封装规则，因为编译结束后没有多个模块，而仅仅只有一个合并后的模块。并且在编译阶段，编译器不知道你将如何使用 `providers` 和动态组件。但是在编译阶段的组件模板解析过程中，编译器知道你是如何使用组件、指令和管道的，所以编译器能够控制它们的私有声明。(`providers` 和 `entryComponents` 整个程序中的动态部分，Angular 编译器不知道它会被如何使用，但是模板中写的组件、指令和管道，是静态部分，Angular 编译器在编译的时候知道它是如何被使用的。这点对理解 Angular 内部工作原理还是比较重要的。)

看一个生成模块工厂的示例，假设有 `A` 和 `B` 两个模块，并且每一个模块都定义了一个 `provider` 和一个 `entryComponents`

```typescript
@NgModule({
  providers: [{provide: 'a', useValue: 'a'}],
  declarations: [AComponent],
  entryComponents: [AComponent]
})
export class AModule {}

@NgModule({
  providers: [{provide: 'b', useValue: 'b'}],
  declarations: [BComponent],
  entryComponents: [BComponent]
})
export class BModule {}
```

根模块 `App` 也定义了一个 `provider` 和根组件 `app`，并导入 `A` 和 `B` 模块

```typescript
@NgModule({
  imports: [AModule, BModule],
  declarations: [AppComponent],
  providers: [{provide: 'root', useValue: 'root'}],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

当编译器编译

