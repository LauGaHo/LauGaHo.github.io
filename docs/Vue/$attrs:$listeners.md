# \$attrs/$listeners

## `$attrs`

包含了父作用域中不被认为 (且不预期为) `props` 的特性绑定 (`class` 和 `style` 除外)，并且可以通过 `v-bind='$attrs'` 传入内部组件。当一个组件没有声明任何 `props` 时，它包含所有父级作用域的绑定 (`class` 和 `style` 除外)。

## `$listeners`

包含了父级作用域中 (不含 `.native` 修饰符) `v-on` 事件监听器。可以通过 `v-on='$listeners'` 传入内部组件。他是一个对象，里面包含了作用在这个组件上的所有事件监听器 (即该组件的父级作用域中的所有监听器)，相当于子组件继承了父组件的事件。

## 例子

### `father.vue`

```vue
<template>
　　 <child :name="name" :age="age" :infoObj="infoObj" @updateInfo="updateInfo" @delInfo="delInfo" />
</template>
<script>
    import Child from '../components/child.vue'

    export default {
        name: 'father',
        components: { Child },
        data () {
            return {
                name: 'Lily',
                age: 22,
                infoObj: {
                    from: '上海',
                    job: 'policeman',
                    hobby: ['reading', 'writing', 'skating']
                }
            }
        },
        methods: {
            updateInfo() {
                console.log('update info');
            },
            delInfo() {
                console.log('delete info');
            }
        }
    }
</script>
```

### `child.vue`

```vue
<template>
    <grand-son :height="height" :weight="weight" @addInfo="addInfo" v-bind="$attrs" v-on="$listeners"  />
    // 通过 $listeners 将父作用域中的事件，传入 grandSon 组件，使其可以获取到 father 中的事件
</template>
<script>
    import GrandSon from '../components/grandSon.vue'
    export default {
        name: 'child',
        components: { GrandSon },
        props: ['name'],
        data() {
          return {
              height: '180cm',
              weight: '70kg'
          };
        },
        created() {
            console.log(this.$attrs); 
　　　　　　　// 结果：age, infoObj, 因为父组件共传来name, age, infoObj三个值，由于name被 props接收了，所以只有age, infoObj属性
            console.log(this.$listeners); // updateInfo: f, delInfo: f
        },
        methods: {
            addInfo () {
                console.log('add info')
            }
        }
    }
</script>
```

### `grandSon.vue`

```vue
<template>
    <div>
        {{ $attrs }} --- {{ $listeners }}
    <div>
</template>
<script>
    export default {
        ... ... 
        props: ['weight'],
        created() {
            console.log(this.$attrs); // age, infoObj, height 
            console.log(this.$listeners) // updateInfo: f, delInfo: f, addInfo: f
            this.$emit('updateInfo') // 可以触发 father 组件中的updateInfo函数
        }
    }
</script>
```



## 总结

结合上方例子中的三段文件：`father.vue`、`child.vue`、`grandSon.vue` 再去使用开头中的两段总结，效果会更佳，祝各位打工人用餐愉快！