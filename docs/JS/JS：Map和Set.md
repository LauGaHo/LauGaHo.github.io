# JS：Map 和 Set

## Map

Map是JS(ES6)的一种字典数据结构，key值不重复，如果有重复，就会覆盖前面的，任何值都可以作为Map的key，包括对象，字符，数字，NaN，symbol。

Map跟Object很像，但是Object只能用string / symbol作为key，Map可以通过size获取键值个数，而Object只能手动计算。

在JS中，NaN === NaN是false，不过在Map中，NaN却被认为是同一个key：

```javascript
const map = new Map();
map.set(NaN, 123);
map.get(NaN); // 123
```

但是对于Object的Key，不同的对象，代表的key值不同：

```javascript
new Map().set({}, 1).set({}, 2).size // 2
```

Map有以下3种创建方式：

```javascript
const emptyMap = new Map();

const map = new Map([
  [1, 'one'],
  [2, 'two'],
  [3, 'three']
]);

const map2 = new Map()
.set(1, 'one')
.set(2, 'two')
.set(3, 'three')
```

复制Map：

```javascript
const original = new Map()
.set(false, 'no')
.set(true, 'yes');

const copy = new Map(original);
```

通过 key 拿到 value：

```javascript
const map = new Map();
map.set('foo', 123);
map.get('foo'); // 123
map.get('bar'); // undefined
```

其他 Map 的方法：

```javascript
const map = new Map();

// .has() checks if a Map has an entry with a given key
map.has('foo') // true

// .delete() remove entries
map.delete('foo');
map.has('foo');// false


const map1 = new Map()
  .set('foo', true)
  .set('bar', false);

// .size retrun the number of entries in a Map
map1.size; //2  

// .clear() removes all entries of a Map
map1.clear();
map1.size; //0


const map2 = new Map()
  .set(false, 'no')
  .set(true, 'yes')

// .keys() returns an iterable over the keys of a Map
for(const key of map2.keys()){
    console.log(key); // output: false true
}

// we can use spreading(...) to convert iterable returned by .keys() to an Array
[... map2.keys()] // [false, true]

// .values() quite like .keys(), but for values instead of keys

//.entries() return an interable over the entries of a Map
for(const entry of map2.entries()) {
    console.log(entry)
    //output:
    //[false, 'no']
    //[ture, 'yes']
}
// we can use sperading(...) to convert iterable returned by .entries() to an Array;
[...map.entries()] // [[false, 'no'], [true, 'yes']]

// we also can use below way to access key and value
for(const [key, value] of map) {
    console.log(key, value);
    // output:
    //false, 'no'
    //true, 'yes'
}

```

只要 Map 只含有 strings 和 symbols 作为 key，那么就可以直接把这个 Map 转为 Object：

```js
const map = new Map([
  ['a', 1],
  ['b', 2]
]);

const obj = Object.fromEntries(map); // { a: 1, b: 2}
```

也可以把 Object 转换为 Map：

```javascript
const obj = {
  a: 1,
  b: 2
}
const map = new Map(Object.entries(obj)); // new Map([['a', 1], ['b', 2]])
```

应用：计算字符串中，字符出现的次数

```javascript
function countChars(chars: string) {
  const charsCounts = new Map();
  for (let ch of chars) {
    ch = ch.toLowerCase();
    const prevCount = charCounts.get(ch);
    charCounts.set(ch, prevCount ? prevCount + 1 : 1);
  }
  return charsCounts;
}
```

如果想要像数组一样，map 和 filter，那么就必须要把 Map 先转化成为数组：

```javascript
const originalMap = new Map()
		.set(1, 'a')
		.set(2, 'b')
		.set(3, 'c')

const mappedMap = new Map( //step 3
		[...originalMap] // step 1
  			.map(([k, v]) => [k * 2, '_' + v]) // step 2
);
// 相当于：new Map([[2, '_a'], [4, '_b'], [6, '_c']])

const filteredMap = new Map( // step 3
		[...originalMap] // step 1
  			.filter(([k, v]) => k < 3) // step 2
)
// 相当于：new Map([[1, 'a'], [2, 'b']])
```

如果想要合并两个 Map：

```javascript
const map1 = new Map()
		.set(1, '1a')
		.set(2, '1b')
		.set(3, '1c');

const map2 = new Map()
		.set(2, '2b')
		.set(3, '2c')
		.set(4, '2d');

const combinedMap = new Map([...map1, ...map2]);
// 相当于：new Map([[1, '1a'], [2, '2b'], [3, '2c'], [4, '2d']])
```

## WeakMap

WeakMap 跟 Map 非常像，只不过多了以下限制：

- WeakMap 就是一个黑盒子

  - 我们不能直接通过 keys / values / entries 来 iterate 或者是 loop WeakMap，并且不能计算它的size。
  - 我们不能清除 WeakMap，如有需要，只能重新创建一个。

- WeakMap 的 key，必须是 objects

  ```javascript
  const wm = new WeakMap();
  wm.set(123, 'test'); // TypeError: Invalid value used as weak map key
  ```

- WeakMap 的 key 是弱饮用

  - 正常来说，如果有对象还被引用，那么就不会被垃圾回收。但是 WeakMap 不一样，在 object 作为 key 的时候，是可以被垃圾回收的，这会导致整个 entry 也被删掉，并且这个没有办法检测到这种行为。

    ```javascript
    const vm = new WeakMap();
    {
      const obj = {};
      vm.set(obj, 'attachedValue'); // (A)
    }
    // (Bc)
    ```

    在 (A) 这一行我们给 obj 这个 key 赋值，但是在 (B) 这一行，obj 这个 entry 就有可能被垃圾回收掉了，但是 vm 还在，并且没有办法手动删掉 vm

  

  应用：

  - 用 WeakMap 来保存计算结果

    ```javascript
    const cache = new WeakMap();
    function countOwnKeys(obj) {
      if (cache.has(obj))  {
        return [cache.get(obj), 'cached'];
      } 
      else {
        const count = Object.keys(obj).length;
        cache.set(obj, count);
        return [count, 'computed'];
      }
    }
    ```

  - 用 WeakMap 来保存 private data

    ```javascript
    const _counter = new WeakMap();
    const _action = new WeakMap();
    
    class Countdown {
        constructor(counter, action) {
            _counter.set(this, counter);
            _action.set(this, action);
        }
        dec() {
            let counter = _counter.get(this);
            counter--;
            _counter.set(this, counter);
            if (counter === 0) {
                _action.get(this)();
            }
        }
    }
    ```

    WeakMap 的方法有：

  - new WeakMap( )

  - .delete( key )

  - .get( key )

  - .has( key )

  - .set( key, value )

## Set

Set 跟数组很像，但是成员的值都是唯一的，没有重复的值，并且 Set 对象允许存储任何类型的值，无论是原始值或者是对象引用。Set函数可以接受一个数组（或者具有 iterable 接口的其他数据结构）作为参数，用来初始化。

有以下三种方式创建 Set：

```javascript
const emptySet = new Set();

const set = new Set(['red', 'green', 'blue']);

const set = new Set()
		.add('red')
		.add('green')
		.add('blue')
```

常用的 Set 方法：

```javascript
// .add() adds an element to a Set
const set = new set();
set.add('red');

// .has() checks if an elements is a member of a Set
set.has('red'); // true

// .delete() removes an element from a Set
set.delete('red'); // true
set.has('red'); // false

// .size contains the number of elements in a Set
const set1 = new Set()
  .add('foo')
  .add('bar')
set1.size; // 2

// .clear() removes all elements of a Set
set1.clear(); 
set1.size; //0

// Iterating over Sets
const set2 = new Set(['red', 'green', 'blue']);
for(const x of set2) {
    console.log(x)
    //outouts:
    //'red'
    //'green',
    //'blue'
}

// use spreading(...) to convert set to array
const set3 = new Set(['red', 'green', 'blue']);
const arr = [...set3]; // ['red', 'green', 'blue']
```

应用

移除数组中的重复项：

```javascript
const set4 = new Set([1, 2, 1, 1, 2, 3, 3, 2, 1]);
const arr = [...set4]; // [1, 2, 3]
```

字符串是 iterable，所以也可以作为 Set 的参数：

```javascript
new Set('abc');
new Set(['a', 'b', 'c']); // 这两个是一样的
```

NaN 对于 Set 来说也是一个值，对于任何 Object 都是不同的值：

```javascript
const set = new Set([NaN, NaN, NaN]);
set.size; // 1

const set1 = new Set([{}, {}]);
set1.size; // 2
```

Union 两个 Set：

```javascript
const a = new Set([1, 2, 3]);
const b = new Set([4, 3, 2]);
const union = new Set([...a, ...b]); // new Set([1, 2, 3, 4])
```

Intersection 两个 Set：

```javascript
const a = new Set([1, 2, 3]);
const b = new Set([4, 3, 2]);
const intersection = new Set(
		[...a]
  	.filter(x => b.has(x))
); // new Set([2, 3])
```

Difference 两个 Set：

```javascript
const a = new Set([1,2,3]);
const b = new Set([4,3,2]);
const difference = new Set(
  [...a]
  .filter(x=> !b.has(x))
); // new Set([1, 4])
```

Mapping over Set:

```javascript
const set = new Set([1,2,3]);
const mappedSet = new Set([...set].map(x=>x*2)); // new Set([2,4,6])
```

Filtering Set:

```javascript
const set = new Set([1,2,3,4,5]);
const filteredSet = new Set([...set].filter(x=>(x%2)===0)); // new Set([2,4])
```

## WeakSet

WeakSet 跟 Set 很像，只不过多了下面的限制：

- WeakSet 是个黑盒子
  - 我们不能直接通过 keys / values / entries 来 iterate 或者 loop WeakMap，并且不能计算它的size。
  - 我们不能清除 WeakSet，如有需要，只能重新创建一个。
- WeakSet 的 key 是弱引用
  - 正常来说，如果有对象还被引用，那么就不会 被垃圾回收。但是 WeakSet 不一样，在 object 作为 key 的时候，是可以被垃圾回收的，这会导致整个 entry 也被删掉，并且这个没有办法检测到这种行为

WeakSet 的方法有：

- new WeakSet( )
- .delete( value )
- .get( value )
- .has( value )
- .set( value )