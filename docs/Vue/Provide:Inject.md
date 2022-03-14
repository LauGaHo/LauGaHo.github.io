# Provide/Inject

## 基本使用

通常，父组件向子组件进行传值时，会使用 `props`。但是如果需要进行传值的组件之间是跨越多个层级的时候，使用 `props` 就会显得格外麻烦，这个时候就可以使用一对 `provide` 和 `inject`。无论组件的层次结构多复杂，父组件都可以为所有子孙组件提供依赖。这个特性：父组件有一个 `provide` 选项来提供数据，子组件有一个 `inject` 选项来使用这些数据。

例如有一个这样的层次结构：

```javascript
Root
	TodoList
  	TodoItem
    TodoListFooter
    	ClearTodosButton
      TodoListStatistics
```

如果要将 `todo-items` 的长度直接传递给 `TodoListStaticstics`，就需要将 `props` 逐级传递下去：`TodoList` -> `TodoListFooter` -> `TodoListStatistics`。通过 `provide` 和 `inject` 的方式，可以直接这样操作：

```vue
const app = Vue.createApp({})

app.component('todo-list', {
  data() {
    return {
      todos: ['Feed a cat', 'Buy tickets']
    }
  },
  provide: {
    user: 'John Doe'
  },
  template: `
    <div>
      {{ todos.length }}
      <!-- 模板的其余部分 -->
    </div>
  `
})

app.component('todo-list-statistics', {
  inject: ['user'],
  created() {
    console.log(`Injected property: ${this.user}`) // > 注入的 property: John Doe
  }
})
```

但是，如果尝试在此处 `provide` 一些组件的实例 `property`，这将是不起作用的：

```vue
app.component('todo-list', {
  data() {
    return {
      todos: ['Feed a cat', 'Buy tickets']
    }
  },
  provide: {
    todoLength: this.todos.length // 将会导致错误 `Cannot read property 'length' of undefined`
  },
  template: `
    ...
  `
})
```

要访问组件实例的 `property`，就需要将 `provide` 转换成返回对象的函数：

```vue
app.component('todo-list', {
  data() {
    return {
      todos: ['Feed a cat', 'Buy tickets']
    }
  },
  provide() {
    return {
      todoLength: this.todos.length
    }
  },
  template: `
    ...
  `
})
```

实际上，可以将依赖注入看做是一个长距离的 `props`，除了：

- 父组件不需要知道哪些子组件使用了它 `provide` 的 `property`
- 子组件不需要知道 `inject` 的 `property` 来自哪里



## 处理响应性

上方例子中，如果更改了 `todos` 的列表，这个变化并不会反映在 `inject` 的 `todoLength` 中。这是因为默认情况下，`provide/inject` 绑定并不是响应式的。我们可以通过传递一个 `ref` 属性或者 `reactive` 对象给 `provide` 来改变这种行为。在上方例子中，如果想对祖先组件中的更改做出响应，就需要为 `provide` 的 `todoLength` 分配一个组合式 API `computed` 属性：

```vue
app.component('todo-list', {
  // ...
  provide() {
    return {
      todoLength: Vue.computed(() => this.todos.length)
    }
  }
})

app.component('todo-list-statistics', {
  inject: ['todoLength'],
  created() {
    console.log(`Injected property: ${this.todoLength.value}`) // > 注入的 property: 5
  }
})
```

在这种情况下，任何对 `todos.length` 的修改都会被正确地反应在注入 `todoLength` 的组件中。