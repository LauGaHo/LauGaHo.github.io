# JS：展开运算符

## 展开运算符 ( . . . )

展开运算符，是 ES6 中的新语法，是把可迭代的对象 ( string, object, 数组 )展开，可以用在函数调用 / 数组构造的时候，将数组表达式 / string 在语法层面展开，还可以将对象表达式按照 key-value 的方式展开。

展开运算符只能用于可迭代对象。

**函数调用：**

假如你把 . . . 放在函数的参数里，就说明这个参数必须是 interable object，然后这个对象就会被展开成为函数的参数

```javascript
function func(x, y) {
  console.log(x);
  console.log(y);
}
const someIterable = ['a', 'b'];
func(...someIterable);
// same as func('a', 'b')
// Output:
// 'a'
// 'b'

> Math.max(-1, 5, 11, 3)
11
> Math.max(...[-1, 5, 11, 3])
11
> Math.max(-1, ...[-5, 11], 3)
11

const arr1 = ['a', 'b'];
const arr2 = ['c', 'd'];
arr1.push(...arr2); // ['a','b','c','d']
```

**数组构造或字符串**

```javascript
const numbers = [1,2,3];
console.log([...numbers, '4', ...'hello', 6];) // Array [1, 2, 3, "4", "h", "e", "l", "l", "o", 6]

var parts = ['shoulders', 'knees'];
var lyrics = ['head', ...parts, 'and', 'toes']; //  ["head", "shoulders", "knees", "and", "toes"]

//数组拷贝
var arr = [1,2,3];
var arr2 = [...arr];
arr2.push(4); // arr2 此时变成了 [1,2,3,4]. arr不受影响

//连接多个数组
var arr1 = [0,1,2];
var arr2 = [3,4,5];
var arr3 = [...arr1, ...arr2]; // [0,1,2,3,4,5]


//对象拷贝（浅拷贝，并且不包含prototype）和合并
var obj1 = {foo:'bar', x:42};
var obj2 = {foo:'baz', y:13};
var cloneObj = {...obj1}; // {foo:'bar', x:42}
var mergeObj = {...obj1, ...obj2}; // {foo:'baz', x:42, y:13}
```

**函数参数收集**

```javascript
function foo (...args) {
  console.log(args);
}

// 类似于 foo([1, 2, 3, 4, 5, 6])
foo(1, 2, 3, 4, 5, 6); // [1, 2, 3, 4, 5, 6]
```

**为对象增加属性**

```javascript
const basicSquirtle = { name: 'Squirtle', type: 'Water' };
const fullSquirtle = {
  ...basicSquirtle,
  species: 'Tiny Turtle',
  evolution: 'Wartortle'
};

console.log(fullSquirtle); 

//Result: { name: 'Squirtle', type: 'Water', species: 'Tiny Turtle', evolution: 'Wartortle' }
```

**复制具有嵌套结构的数据 / 对象**

```javascript
const pokemon = {
  name: 'Squirtle',
  type: 'Water',
  abilities: ['Torrent', 'Rain Dish']
};

const squirtleClone = { ...pokemon };

pokemon.name = 'Charmander';
pokemon.abilities.push('Surf');

console.log(squirtleClone); 

//Result: { name: 'Squirtle', type: 'Water', abilities: [ 'Torrent', 'Rain Dish', 'Surf' ] }
```

当我们修改原对象的 name 属性时，我们的克隆对象的 name 属性没有受到影响，这个符合预期。

但是当我们修改原对象的 abilities 属性时，我们的克隆对象也被修改了。

原因很简单，因为复制过来的 abilities 时一个引用类型，原数据改了，用到它的地方也会跟着改。

知道原因，再解决就很简单了

**复制引用类型的数据**

```javascript
const pokemon = {
  name: 'Squirtle',
  type: 'Water',
  abilities: ['Torrent', 'Rain Dish']
};

const squirtleClone = { ...pokemon, abilities: [...pokemon.abilities] };

pokemon.name = 'Charmander';
pokemon.abilities.push('Surf');

console.log(squirtleClone); 

//Result: { name: 'Squirtle', type: 'Water', abilities: [ 'Torrent', 'Rain Dish' ] }
```

**增加条件属性**

```javascript
const pokemon = {
  name: 'Squirtle',
  type: 'Water'
};

const abilities = ['Torrent', 'Rain dish'];
const fullPokemon = abilities ? { ...pokemon, abilities } : pokemon;

console.log(fullPokemon);
```

**短路**

```javascript
const pokemon = {
  name: 'Squirtle',
  type: 'Water'
};

const abilities = ['Torrent', 'Rain dish'];
const fullPokemon = {
  ...pokemon,
  ...(abilities && { abilities })
};

console.log(fullPokemon);
```

如果 abilities 为 true

```javascript
const fullPokemon = {
  ...pokemon,
  ...{ abilities }
}
```

