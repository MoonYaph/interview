### 变量对象

我们在创建应用程序的时候，总免不了要声明变量和函数。那么，当我们需要使用这些东西的时候，解释器(interpreter)是怎么样、从哪里找到我们的数据(函数，变量)的，这个过程究竟发生了什么呢？

大部分ECMAScript程序员应该都知道变量与 执行上下文 密切相关：
```javascript
var x = 10; //variable of global object
(function (){
  var b = 20; // local variable of the function context
})()

alert(a); // 10
alert(b); // b is not defined 
```

基于当前版本的规范，独立作用域只能通过“函数(function)”代码类型的执行上下文创建。那么，想对于C/C++举例来说，ECMAScript里， for 循环并不能创建一个局部的上下文。(译者注：就是局部作用域)：

```javascript
for (var k in {a; 1, b: 2}){
  alert(k)
}
alert(K) // variable "k" still in scope even the loop is finished
```

#### 数据声明

如果变量与执行上下文相关，那么它自己应该知道它的数据存储在哪里和如何访问。这种机制被称作 变量对象(variable object).

变量对象 (缩写为VO)就是与执行上下文相关的对象(译者注：这个“对象”的意思就是指某个东西)，它存储下列内容：
- 变量(var, variableDeclaration)
- 函数声明 (FunctionDecalration, FD)
- 函数的形参

以上均在上下文中声明

一个变量对象完全可能用正常的ECMAScript对象的形式来表现：

```javascript
VO = {};
```

VO就是执行上下文的属性
```javascript
activeExecutionContext = {
  VO: {
    // context data { var, FD}
  }
}
```

只有全局上下文的变量对象允许通过VO的属性名称简介访问(因为在全局上下文里，全局对象自身就是变量对象)，在其他上下文中是不可能直接访问到VO的，因为变量对象是实现机制内部的事情。
当我们声明一个变量或一个函数的时候，同时还用变量的名称和值，在VO里创建了一个新的属性。
```javascript
var a = 10;
function test(x) {
  var v = 20
}
test(30)
```
对应的变量对象是：

```javascript
// Variable object of the global context
VO(globalContext) = {
  a : 10,
  test,
};

// Variable object of the test

VO(test, functionContext) = {
  x: 30,
  b: 20
}
```

#### 不同执行上下文中的变量对象

对于所有类型的执行上下文来说，变量对象的一些操作(如变量初始化)和行为都是共通的。从这个角度来看，把变量对象作为抽象的基本事物来理解更容易。而在函数上下文里同样可以通过变量对象定义一些相关的额外细节。

```js
AbstractVO(generic behavior of the variable instantiation process)
  GlobalContextVO // VO===this===global
  FunctionContextVO // VO===AO <arguments> 
```
#### 全局上下文中的变量对象

全局对象实在进入任何执行上下文之前就已经创建的对象；这个对象只存在一份，它的属性在程序中任何地方都可以访问，全局对象的生命周期终止于程序退出那一刻。

初始创建阶段，全局对象通过Math, String, Date, parseInt等属性初始化，同样可以附加其他对象作为属性，其中包括可以引用全局对象自身的对象。例如，在DOM中，全局对象的window属性就是引用全局对象自身的属性
```js
global = [
  Math:<...>
  String: <...>
  window: global
]
```

因为全局对象是不能通过名称直接访问的，所以当访问全局对象的属性时，通常忽略前缀。尽管如此，通过全局上下文的this还是有可能直接访问到全局对象的，同样也可以通过引用自身的属性来访问，例如，DOM中的window。综上所述，代码可以简写为

```js
String(10); // means global.String(10);
 
// with prefixes
window.a = 10; // === global.window.a = 10 === global.a = 10;
this.b = 20; // global.b = 20;
```

因此，全局上下文中的变量对象就是全局对象自身(global object itself):
```js
VO(globalContext) === global;
```

准确理解“全局上下文中的变量对象就是全局对象自身”是非常必要的，基于这个事实，在全局上下文中声明一个变量时，我们才能够通过全局对象的属性间接访问到这个变量(例如，当事先未知变量名时)
```js
var a = new String('test');
 
alert(a); // directly, is found in VO(globalContext): "test"
 
alert(window['a']); // indirectly via global === VO(globalContext): "test"
alert(a === this.a); // true
 
var aKey = 'a';
alert(window[aKey]); // indirectly, with dynamic property name: "test"
```

#### 函数上下文中的变量对象

在函数执行上下文，VO是不能直接访问的，此时由激活对象的qrguments属性初始化，argumens属性的值是argument object

```js
AO = {
  arguments: <ArgO>
};
```
Arguments objects 是函数上下文里的激活对象中的内部对象，它包括下列属性：

- callee — 指向当前函数的引用；
- length — 真正传递的参数的个数；
- properties-indexes (字符串类型的整数) 属性的值就是函数的参数值(按参数列表从左到右排列)。 properties-indexes内部元素的个数等于arguments.length. properties-indexes 的值和实际传递进来的参数之间是共享的。(译者注：共享与不共享的区别可以对比理解为引用传递与值传递的区别)
例如：

```js
function foo(x, y, z) {
 
  alert(arguments.length); // 2 – quantity of passed arguments
  alert(arguments.callee === foo); // true
 
  alert(x === arguments[0]); // true
  alert(x); // 10
 
  arguments[0] = 20;
  alert(x); // 20
 
  x = 30;
  alert(arguments[0]); // 30
 
  // however, for not passed argument z,
  // related index-property of the arguments
  // object is not shared
 
  z = 40;
  alert(arguments[2]); // undefined
 
  arguments[2] = 50;
  alert(z); // 40
 
}
 
foo(10, 20);
```
#### 分阶段处理上下文代码

执行上下文代码分为两个阶段：
* 进入执行上下文
* 执行代码

#### 进入执行上下文

当进入执行上下文(代码执行前)时，VO已被下列属性填充
 - 函数的所有形式参数(如果我们是在函数执行上下文中)
    - 变量对象的一个属性，这个属性有一个形式参数的名称和值组成，如果没有对应传递实际参数，那么这个属性就由形式参数的名称和undefined值组成；
 - 所有函数声明(FD)
    - 变量对象的一个属性，这个属性由一个函数对象的名称和值组成；如果变量对象已经存在相同名称的属性，则完全替换这个属性
- 所有变量声明(var,VariabelDeclaration)
    - 变量对象的一个属性，这个属性由变量名称和undefined值组成，如果变量名称跟已经声明的形式参数和函数相同。则变量声明不会干扰已经存在的这类属性

```js
function test(a, b){
  var c = 10;
  function d() {}
  var e = function _e(){}
  (function x(){})()
}
test(10) // call
```
当进入test函数的上下文时(传递参数10),AO如下：
```js
AO(test) = {
  a: 10,
  b: undefined,
  c: undefined,
  e: unefined,
  d: <reference to FunctionDeclartation 'd'>,
}
```

>注意，AO里并不包含函数“x”。这是因为“x” 是一个函数表达式(FunctionExpression, 缩写为 FE) 而不是函数声明，函数表达式不会影响VO(这里的VO指的就是AO)。 不管怎样，函数“_e” 同样也是函数表达式，但是就像我们下面将看到的那样，因为它分配给了变量 “e”，所以它变成可以通过名称“e”来访问。 FunctionDeclaration 与 FunctionExpression 的不同，

#### 执行代码

这一刻，AO/VO已经被属性(不过，并不是所有的属性都有值，大部分属性的值还是系统默认的初始值undefined )填满。AO/VO在代码解释期间被修改如下：

```js
AO['c'] = 10;
AO['e'] = <reference to FunctionExpression '_e'>
```
>再次注意，因为FunctionExpression“_e”保存到了已声明的变量“e”上，所以它仍然存在于内存中(就是还在AO/VO中的意思)。而FunctionExpression。未保存的函数表达式只有在它自己的定义或递归中才能被调用。 “x” 并不存在于AO/VO中。即，如果我们想尝试调用“x”函数，不管在函数定义之前还是之后，都会出现一个错误“x is not defined”

```js
alert(x); // function
 
var x = 10;
alert(x); // 10
 
x = 20;
 
function x() {};
 
alert(x); // 20
```

为什么第一个alert “x” 的返回值是function，而且它还是在“x” 声明之前访问的“x” 的？为什么不是10或20呢？因为，根据规范 — 当进入上下文时，往VO里填入函数声明；在相同的阶段，还有一个变量声明“x”，那么正如我们在上一个阶段所说，变量声明在顺序上跟在函数声明和形式参数声明之后，而且，在这个阶段(这个阶段是指进入执行上下文阶段)，变量声明不会干扰VO中已经存在的同名函数声明或形式参数声明，因此，在进入上下文时，VO的结构如下：

```js
VO = {};
VO['x'] = <reference to FunctionDeclaration 'x'>;
// found var x = 10
// if function 'x' would not be already defined
// then 'x' be undefined, but in our case
// variable declaraion does not disturb the value of the function with the same name
VO['x'] = <the value is not disturbed still function>
```
随后在执行代码阶段，VO做如下修改：
```js
VO['x'] = 10;
VO['x'] = 20;
```

在下面的列子中我们可以再次看到，变量是在进入上下文阶段放入VO中，因为，虽然else部分永远不会执行，但是不管怎么样，变量b仍然存在VO中，其值时undefined
```js
if(true) {
  var a = 1;
} else {
  var b = 2
}
alert(a) // 1
alert(b) // undefined, not b is not defined
```

#### 关于变量
通常，各类文章和JavaScript相关的书籍都声称：“不管是使用var关键字(在全局上下文)还是不使用var关键字(在任何地方)，都可以声明一个变量”。请记住，这绝对是谣传：

>任何时候，变量只能通过使用var关键字才能声明。

那么向下面这样分配：
```js
a = 10;
```
这不仅给全局对象创建了一个新属性(但是它不是变量)。不是变量的意思并不是说它不能被改变，而是指它不符合ECMAScript规范中的变量概念，所以它“不是变量”(它之所以能成为全局对象的属性，完全是因为VO(globalContext) === global，大家还记得这个吧？)。

```js
alert(a); // undefined
alert(b); // "b" is not defined
 
b = 10;
var a = 20;
```
所有根源仍然是VO和它的修改阶段(进入上下文 阶段和执行代码 阶段)：
进入上下文阶段：
```js
VO = {
  a: undefined
}
```
我们可以看到，因为“b”不是一个变量，所以在这个阶段根本就没有“b”，“b”将只在执行代码阶段才会出现(但是在我们这个例子里，还没有到那就已经出错了)。

```js
alert(a); // undefined, we know why
 
b = 10;
alert(b); // 10, created at code execution
 
var a = 20;
alert(a); // 20, modified at code execution
```

关于变量，还有一个重要的知识点。变量相对于简单属性来说，变量有一个特性(attribute)：{DontDelete},这个特性的含义就是不同通过delete操作符直接删除变量属性。

```js
a = 10;
alert(window.a); // 10
 
alert(delete a); // true
 
alert(window.a); // undefined
 
var b = 20;
alert(window.b); // 20
 
alert(delete b); // false
 
alert(window.b); // still 20
```

但是，在eval上下文，这个规则并不起作用，因为在这个上下文里，变量没有{DontDelete}特性。

```js
eval('var a = 10;');
alert(window.a); // 10
 
alert(delete a); // true
 
alert(window.a); // undefined
```