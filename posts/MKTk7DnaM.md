---
title: 'ES6新语法'
date: 2022-03-01 10:22:06
tags: [ES6]
published: true
hideInList: false
feature: /post-images/MKTk7DnaM.jpg
isTop: false
---
### const 与 let 变量
使用var带来的麻烦:

```js
function getClothing(isCold) {
  if (isCold) {
    var freezing = 'Grab a jacket!';
  } else {
    var hot = 'It's a shorts kind of day.';
    console.log(freezing);
  }
}
```

运行getClothing(false)后输出的是undefined,这是因为执行function函数之前,所有变量都会被提升, 提升到函数作用域顶部.

let与const声明的变量解决了这种问题,因为他们是块级作用域, 在代码块(用{}表示)中使用let或const声明变量, 该变量会陷入暂时性死区直到该变量的声明被处理.

```js
function getClothing(isCold) {
  if (isCold) {
    const freezing = 'Grab a jacket!';
  } else {
    const hot = 'It's a shorts kind of day.';
    console.log(freezing);
  }
}
```


运行getClothing(false)后输出的是ReferenceError: freezing is not defined,因为 freezing 没有在 else 语句、函数作用域或全局作用域内声明，所以抛出 ReferenceError。

**关于使用let与const规则:**

使用let声明的变量可以重新赋值,但是不能在同一作用域内重新声明
使用const声明的变量必须赋值初始化,但是不能在同一作用域类重新声明也无法重新赋值.



### 模板字面量
在ES6之前,将字符串连接到一起的方法是+或者concat()方法,如

```js
const student = {
  name: 'Richard Kalehoff',
  guardian: 'Mr. Kalehoff'
};
const teacher = {
  name: 'Mrs. Wilson',
  room: 'N231'
}
let message = student.name + ' please see ' + teacher.name + ' in ' + teacher.room + ' to pick up your report card.';
```

模板字面量本质上是包含嵌入式表达式的字符串字面量.
模板字面量用倒引号 ( `` )（而不是单引号 ( '' ) 或双引号( "" )）表示，可以包含用 ${expression} 表示的占位符

```js
let message = `${student.name} please see ${teacher.name} in ${teacher.room} to pick up your report card.`;
```

### 解构
在ES6中,可以使用解构从数组和对象提取值并赋值给独特的变量

解构数组的值:

```js
const point = [10, 25, -34];
const [x, y, z] = point;
console.log(x, y, z);
Prints: 10 25 -34
```

[]表示被解构的数组, x,y,z表示要将数组中的值存储在其中的变量, 在解构数组是, 还可以忽略值, 例如const[x,,z]=point,忽略y坐标.

解构对象中的值:

```js
const gemstone = {
  type: 'quartz',
  color: 'rose',
  karat: 21.29
};
const {type, color, karat} = gemstone;
console.log(type, color, karat);
```


花括号 { } 表示被解构的对象，type、color 和 karat 表示要将对象中的属性存储到其中的变量

对象字面量简写法

```js
let type = 'quartz';
let color = 'rose';
let carat = 21.29;
const gemstone = {
  type: type,
  color: color,
  carat: carat
};
console.log(gemstone);
```


使用和所分配的变量名称相同的名称初始化对象时如果属性名称和所分配的变量名称一样，那么就可以从对象属性中删掉这些重复的变量名称。

```js
let type = 'quartz';
let color = 'rose';
let carat = 21.29;
const gemstone = {type,color,carat};
console.log(gemstone);
```


简写方法的名称:

```js
const gemstone = {
  type,
  color,
  carat,
  calculateWorth: function() {
    // 将根据类型(type)，颜色(color)和克拉(carat)计算宝石(gemstone)的价值
  }
};
```


匿名函数被分配给属性 calculateWorth，但是真的需要 function 关键字吗？在 ES6 中不需要！

```js
let gemstone = {
  type,
  color,
  carat,
  calculateWorth() { ... }
};
```

### for...of循环
for...of循环是最新添加到 JavaScript 循环系列中的循环。
它结合了其兄弟循环形式 for 循环和 for...in 循环的优势，可以循环任何可迭代（也就是遵守可迭代协议）类型的数据。默认情况下，包含以下数据类型：String、Array、Map 和 Set，注意不包含 Object 数据类型（即 {}）。默认情况下，对象不可迭代。

#### for循环

```js
const digits = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
for (let i = 0; i < digits.length; i++) {
  console.log(digits[i]);
}
```

for 循环的最大缺点是需要跟踪计数器和退出条件。
虽然 for 循环在循环数组时的确具有优势，但是某些数据结构不是数组，因此并非始终适合使用 loop 循环。

#### for...in循环

```js
const digits = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
for (const index in digits) {
  console.log(digits[index]);
}
```

依然需要使用 index 来访问数组的值
当你需要向数组中添加额外的方法（或另一个对象）时，for...in 循环会带来很大的麻烦。因为 for...in 循环循环访问所有可枚举的属性，意味着如果向数组的原型中添加任何其他属性，这些属性也会出现在循环中。

```js
Array.prototype.decimalfy = function() {
  for (let i = 0; i < this.length; i++) {
    this[i] = this[i].toFixed(2);
  }
};
const digits = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
for (const index in digits) {
  console.log(digits[index]);
}
```

#### forEach 循环 

forEach 循环是另一种形式的 JavaScript 循环。但是，forEach() 实际上是数组方法，因此只能用在数组中。也无法停止或退出 forEach 循环。如果希望你的循环中出现这种行为，则需要使用基本的 for 循环。

#### for...of循环
for...of 循环用于循环访问任何可迭代的数据类型。
for...of 循环的编写方式和 for...in 循环的基本一样，只是将 in 替换为 of，可以忽略索引。

```js
const digits = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
for (const digit of digits) {
  console.log(digit);
}
```


建议使用复数对象名称来表示多个值的集合。这样，循环该集合时，可以使用名称的单数版本来表示集合中的单个值。例如，for (const button of buttons) {…}。

for...of 循环还具有其他优势，解决了 for 和 for...in 循环的不足之处。你可以随时停止或退出 for...of 循环。

```js
for (const digit of digits) {
  if (digit % 2 === 0) {
    continue;
  }
  console.log(digit);
}
```


不用担心向对象中添加新的属性。for...of 循环将只循环访问对象中的值。

```js
Array.prototype.decimalfy = function() {
  for (i = 0; i < this.length; i++) {
    this[i] = this[i].toFixed(2);
  }
};
const digits = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];
for (const digit of digits) {
  console.log(digit);
}

```

### 展开运算符
展开运算符（用三个连续的点 (...) 表示）是 ES6 中的新概念，使你能够将字面量对象展开为多个元素

```js
const books = ["Don Quixote", "The Hobbit", "Alice in Wonderland", "Tale of Two Cities"];
console.log(...books);
Prints: Don Quixote The Hobbit Alice in Wonderland Tale of Two Cities
```

展开运算符的一个用途是结合数组。

如果你需要结合多个数组，在有展开运算符之前，必须使用 Array的 concat() 方法。

```js
const fruits = ["apples", "bananas", "pears"];
const vegetables = ["corn", "potatoes", "carrots"];
const produce = fruits.concat(vegetables);
console.log(produce);
Prints: ["apples", "bananas", "pears", "corn", "potatoes", "carrots"]
```

使用展开符来结合数组

```js
const fruits = ["apples", "bananas", "pears"];
const vegetables = ["corn", "potatoes", "carrots"];
const produce = [...fruits,...vegetables];
console.log(produce);
```

**剩余参数(可变参数)**
使用展开运算符将数组展开为多个元素, 使用剩余参数可以将多个元素绑定到一个数组中.
剩余参数也用三个连续的点 ( ... ) 表示，使你能够将不定数量的元素表示为数组.

**用途1: 将变量赋数组值时:**

```js
const order = [20.17, 18.67, 1.50, "cheese", "eggs", "milk", "bread"];
const [total, subtotal, tax, ...items] = order;
console.log(total, subtotal, tax, items);
```

**用途2: 可变参数函数**
对于参数不固定的函数,ES6之前是使用参数对象(arguments)处理:

```js
function sum() {
  let total = 0;  
  for(const argument of arguments) {
    total += argument;
  }
  return total;
}
```

在ES6中使用剩余参数运算符则更为简洁,可读性提高:

```js
function sum(...nums) {
  let total = 0;  
  for(const num of nums) {
    total += num;
  }
  return total;
}
```


### ES6箭头函数
ES6之前,使用普通函数把其中每个名字转换为大写形式：

```js
const upperizedNames = ['Farrin', 'Kagure', 'Asser'].map(function(name) { 
  return name.toUpperCase();
});
```

箭头函数表示:

```js
const upperizedNames = ['Farrin', 'Kagure', 'Asser'].map(
  name => name.toUpperCase()
);
```

普通函数可以是函数声明或者函数表达式, 但是箭头函数始终都是表达式, 全程是箭头函数表达式, 因此因此仅在表达式有效时才能使用，包括：

存储在变量中，
当做参数传递给函数，
存储在对象的属性中。

```js
const greet = name => `Hello ${name}!`;
```


可以如下调用:

```js
greet('Asser');
```


如果函数的参数只有一个,不需要使用()包起来,但是只有一个或者多个, 则必须需要将参数列表放在圆括号内:

```js
// 空参数列表需要括号
const sayHi = () => console.log('Hello Udacity Student!');
// 多个参数需要括号
const orderIceCream = (flavor, cone) => console.log(`Here's your ${flavor} ice cream in a ${cone} cone.`);
orderIceCream('chocolate', 'waffle');
```


一般箭头函数都只有一个表达式作为函数主题:

```js
const upperizedNames = ['Farrin', 'Kagure', 'Asser'].map(
  name => name.toUpperCase()
);
```


这种函数表达式形式称为简写主体语法:

1,在函数主体周围没有花括号,
2,自动返回表达式
3,但是如果箭头函数的主体内需要多行代码, 则需要使用常规主体语法:

它将函数主体放在花括号内
需要使用 return 语句来返回内容。

```js
const upperizedNames = ['Farrin', 'Kagure', 'Asser'].map( name => {
  name = name.toUpperCase();
  return `${name} has ${name.length} characters in their name`;
});

```

### javascript标准函数this
**new 对象**

```js
const mySundae = new Sundae('Chocolate', ['Sprinkles', 'Hot Fudge']);
```

sundae这个构造函数内的this的值是实例对象, 因为他使用new被调用.

**指定的对象**

```js
const result = obj1.printName.call(obj2);
```

函数使用call/apply被调用,this的值指向指定的obj2,因为call()第一个参数明确设置this的指向

**上下文对象**

```js
data.teleport();
```

函数是对象的方法, this指向就是那个对象,此处this就是指向data.

**全局对象或 undefined**

```js
teleport();
```


此处是this指向全局对象,在严格模式下,指向undefined.

javascript中this是很复杂的概念, 要详细判断this,请参考this豁然开朗

#### 箭头函数和this
对于普通函数, this的值基于函数如何被调用, 对于箭头函数,this的值基于函数周围的上下文, 换句话说,this的值和函数外面的this的值是一样的.

```js
function IceCream() {
    this.scoops = 0;
}
// 为 IceCream 添加 addScoop 方法
IceCream.prototype.addScoop = function() {
    setTimeout(function() {
        this.scoops++;
        console.log('scoop added!');
        console.log(this.scoops); // undefined+1=NaN
        console.log(dessert.scoops); //0
    }, 500);
};
```

```js
const dessert = new IceCream();
dessert.addScoop();
```

传递给 setTimeout() 的函数被调用时没用到 new、call() 或 apply()，也没用到上下文对象。意味着函数内的 this 的值是全局对象，不是 dessert 对象。实际上发生的情况是，创建了新的 scoops 变量（默认值为 undefined），然后递增（undefined + 1 结果为 NaN）;

解决此问题的方式之一是使用闭包(closure):

```js
// 构造函数
function IceCream() {
  this.scoops = 0;
}
// 为 IceCream 添加 addScoop 方法
IceCream.prototype.addScoop = function() {
  const cone = this; // 设置 `this` 给 `cone`变量
  setTimeout(function() {
    cone.scoops++; // 引用`cone`变量
    console.log('scoop added!'); 
    console.log(dessert.scoops);//1
  }, 0.5);
};
const dessert = new IceCream();
dessert.addScoop();
```


箭头函数的作用正是如此, 将setTimeOut()的函数改为剪头函数:

```js
// 构造函数
function IceCream() {
  this.scoops = 0;
}
// 为 IceCream 添加 addScoop 方法
IceCream.prototype.addScoop = function() {
  setTimeout(() => { // 一个箭头函数被传递给setTimeout
    this.scoops++;
    console.log('scoop added!');
    console.log(dessert.scoops);//1
  }, 0.5);
};
const dessert = new IceCream();
dessert.addScoop();
```


**默认参数函数**

```js
function greet(name, greeting) {
  name = (typeof name !== 'undefined') ?  name : 'Student';
  greeting = (typeof greeting !== 'undefined') ?  greeting : 'Welcome';
  return `${greeting} ${name}!`;
}
greet(); // Welcome Student!
greet('James'); // Welcome James!
greet('Richard', 'Howdy'); // Howdy Richard!

```

greet() 函数中混乱的前两行的作用是什么？它们的作用是当所需的参数未提供时，为函数提供默认的值。但是看起来很麻烦, ES6引入一种新的方式创建默认值, 他叫默认函数参数:

```js
function greet(name = 'Student', greeting = 'Welcome') {
  return `${greeting} ${name}!`;
}
greet(); // Welcome Student!
greet('James'); // Welcome James!
greet('Richard', 'Howdy'); // Howdy Richard!

```

### 默认值与解构
**默认值与解构数组**

```js
function createGrid([width = 5, height = 5]) {
  return `Generates a ${width} x ${height} grid`;
}
createGrid([]); // Generates a 5 x 5 grid
createGrid([2]); // Generates a 2 x 5 grid
createGrid([2, 3]); // Generates a 2 x 3 grid
createGrid([undefined, 3]); // Generates a 5 x 3 grid
```

createGrid() 函数预期传入的是数组。它通过解构将数组中的第一项设为 width，第二项设为 height。如果数组为空，或者只有一项，那么就会使用默认参数，并将缺失的参数设为默认值 5。

但是存在一个问题:

```js
createGrid(); // throws an error
Uncaught TypeError: Cannot read property 'Symbol(Symbol.iterator)' of undefined
```

出现错误，因为 createGrid() 预期传入的是数组，然后对其进行解构。因为函数被调用时没有传入数组，所以出现问题。但是，我们可以使用默认的函数参数！

```js
function createGrid([width = 5, height = 5] = []) {
  return `Generating a grid of ${width} by ${height}`;
}
createGrid(); // Generates a 5 x 5 grid
Returns: Generates a 5 x 5 grid
```

**默认值与解构函数**
就像使用数组默认值解构数组一样，函数可以让对象成为一个默认参数，并使用对象解构：

```js
function createSundae({scoops = 1, toppings = ['Hot Fudge']}={}) {
  const scoopText = scoops === 1 ? 'scoop' : 'scoops';
  return `Your sundae has ${scoops} ${scoopText} with ${toppings.join(' and ')} toppings.`;
}
createSundae({}); // Your sundae has 1 scoop with Hot Fudge toppings.
createSundae({scoops: 2}); // Your sundae has 2 scoops with Hot Fudge toppings.
createSundae({scoops: 2, toppings: ['Sprinkles']}); // Your sundae has 2 scoops with Sprinkles toppings.
createSundae({toppings: ['Cookie Dough']}); // Your sundae has 1 scoop with Cookie Dough toppings.
createSundae(); // Your sundae has 1 scoop with Hot Fudge toppings.
```

**数组默认值与对象默认值**
默认函数参数只是个简单的添加内容，但是却带来很多便利！与数组默认值相比，对象默认值具备的一个优势是能够处理跳过的选项。看看下面的代码：

```js
function createSundae({scoops = 1, toppings = ['Hot Fudge']} = {}) { … }
```


在 createSundae() 函数使用对象默认值进行解构时，如果你想使用 scoops 的默认值，但是更改 toppings，那么只需使用 toppings 传入一个对象：

```js
createSundae({toppings: ['Hot Fudge', 'Sprinkles', 'Caramel']});
```


将上述示例与使用数组默认值进行解构的同一函数相对比。

```js
function createSundae([scoops = 1, toppings = ['Hot Fudge']] = []) { … }
```


对于这个函数，如果想使用 scoops 的默认数量，但是更改 toppings，则必须以这种奇怪的方式调用你的函数：

```js
createSundae([undefined, ['Hot Fudge', 'Sprinkles', 'Caramel']]);
```


因为数组是基于位置的，我们需要传入 undefined 以跳过第一个参数（并使用默认值）来到达第二个参数。

### Javascript类
**ES5创建类:**

```js
function Plane(numEngines) {
  this.numEngines = numEngines;
  this.enginesActive = false;
}
// 由所有实例 "继承" 的方法
Plane.prototype.startEngines = function () {
  console.log('starting engines...');
  this.enginesActive = true;
};
```

**ES6创建类**
ES6类只是一个语法糖,原型继续实际上在底层隐藏起来, 与传统类机制语言有些区别.

```js
class Plane {
  //constructor方法虽然在类中,但不是原型上的方法,只是用来生成实例的.
  constructor(numEngines) {
    this.numEngines = numEngines;
    this.enginesActive = false;
  }
  //原型上的方法, 由所有实例对象共享.
  startEngines() {
    console.log('starting engines…');
    this.enginesActive = true;
  }
}
console.log(typeof Plane); //function

```

javascript中类其实只是function, 方法之间不能使用,,不用逗号区分属性和方法.

### 静态方法
要添加静态方法，请在方法名称前面加上关键字 static

```js
class Plane {
  constructor(numEngines) {
    this.numEngines = numEngines;
    this.enginesActive = false;
  }
  static badWeather(planes) {
    for (plane of planes) {
      plane.enginesActive = false;
    }
  }
  startEngines() {
    console.log('starting engines…');
    this.enginesActive = true;
  }
}
```


关键字class带来其他基于类的语言的很多思想,但是没有向javascript中添加此功能
javascript类实际上还是原型继承
创建javascript类的新实例时必须使用new关键字
### super 和 extends
使用新的super和extends关键字扩展类:

```js
class Tree {
  constructor(size = '10', leaves = {spring: 'green', summer: 'green', fall: 'orange', winter: null}) {
    this.size = size;
    this.leaves = leaves;
    this.leafColor = null;
  }
  changeSeason(season) {
    this.leafColor = this.leaves[season];
    if (season === 'spring') {
      this.size += 1;
    }
  }
}
class Maple extends Tree {
  constructor(syrupQty = 15, size, leaves) {
    super(size, leaves); //super用作函数
    this.syrupQty = syrupQty;
  }
  changeSeason(season) {
    super.changeSeason(season);//super用作对象
    if (season === 'spring') {
      this.syrupQty += 1;
    }
  }
  gatherSyrup() {
    this.syrupQty -= 3;
  }
}
```


使用ES5编写同样功能的类:

```js
function Tree(size, leaves) {
  this.size = size || 10;
  this.leaves = leaves || {spring: 'green', summer: 'green', fall: 'orange', winter: null};
  this.leafColor;
}
Tree.prototype.changeSeason = function(season) {
  this.leafColor = this.leaves[season];
  if (season === 'spring') {
    this.size += 1;
  }
}
function Maple (syrupQty, size, leaves) {
  Tree.call(this, size, leaves);
  this.syrupQty = syrupQty || 15;
}
Maple.prototype = Object.create(Tree.prototype);
Maple.prototype.constructor = Maple;
Maple.prototype.changeSeason = function(season) {
  Tree.prototype.changeSeason.call(this, season);
  if (season === 'spring') {
    this.syrupQty += 1;
  }
}
Maple.prototype.gatherSyrup = function() {
  this.syrupQty -= 3;
}
```

super 必须在 this 之前被调用

在子类构造函数中，在使用 this 之前，必须先调用超级类。

```js
class Apple {}
class GrannySmith extends Apple {
  constructor(tartnessLevel, energy) {
    this.tartnessLevel = tartnessLevel; // 在 'super' 之前会抛出一个错误！
    super(energy); 
  }
}
```



