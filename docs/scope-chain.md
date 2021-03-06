### [作用域链](http://dmitrysoshnikov.com/ecmascript/chapter-4-scope-chain/)

在[变量对象](/docs/variable-object.md)， 已经介绍过执行上下文的数据是以变量对象的属性的形式进行存储的。
还介绍了，每次进入执行上下文的时候，就会创建变量对象，并且赋予其属性初始值，随后在执行代码阶段会对属性值进行更新。
本文要与执行上下文密切相关的另外一个重要的概念——作用域链（Scope Chain）。

众所周知，ECMAScript允许创建内部函数，甚至可以将这些内部函数作为父函数的返回值。
```js
var x = 10
function foo(){
  var y = 20
  function bar() {
    alert(x + y)
  }
  return bar
}
foo()() // 30
```
每个上下文都有自己的变量对象，对于全局上下文而言，其变量对象就是全局对象本身，对于函数而言，器变量对象就是活动对象(AO)
作用域链其实就是所有内部上下文的变量对象的列表。用于变量查询。比如，上述例子中，bar上下文的作用域链包含了AO(bar), AO(foo)和VO(global)

>作用域链是一条变量对象的链，它和执行上下文有关，用于在处理标识符时候进行变量查询。

函数上下文的作用域链在函数调用的时候创建出来，它包含了活跃对象和该函数的内部[[Scope]]属性。关于[[Scope]]会在后面作详细介绍。

```js
activeExecutionContext = {
  Vo:{...}, // AO
  this: thisValue,
  Scope: [
    // 所有域链 所有变量对象的列表 用于标识符查询
  ]
}
```
上述代码中的Scope定义为如下所示：
```js
Scope = AO + [[Scope]]
```
针对我们的例子来说，可以将Scope和[[Scope]用普通的数组来表示

```js
var Scope = [VO1,VO@, VO3,...,VOn] //作用域链
```
除此之外，还可以用分层对象链的数据结构来表示，链中每一个链接都有对父作用域 的引用

```js
var VO1 = {__parent__: null, ... other data}; -->
var VO2 = {__parent__: VO1, ... other data}; -->
// etc.
```

然而，使用数组来表示作用域链会更方便，因此，我们这里就采用数组的表示方式。 除此之外，不论在实现层是否采用包含__parent__特性的分层对象链的数据结构，标准自身对其做了抽象的定义“作用域链是一个对象列表”。 数组就是实现列表这一概念最好的选择。

面将要介绍的 AO+[[Scope]]以及标识符的处理方式，都和函数的生命周期有关。

#### 函数的声明周期
函数的声明周期分为创建阶段和激活调用阶段

####函数的创建
在进入上下文阶段，函数声明会存储在变量/活跃对象中（VO/AO）。让我们来看一个全局上下文中变量声明和函数声明的例子（这种情况下，变量对象就是全局对象本身，应该还没忘记吧？）：

```js
var x =10
function foo() {
  var y = 20
  alert(x + y)
}
foo() // 30
```
在说当前上下文的变量对象前。上述代码中我们看到变量“y”是在“foo”函数中定义的（意味着它存储在“foo”上下文的AO对象中）， 然而变量“x”则并没有在“foo”上下文中定义，自然也不会添加到“foo”的AO中。乍一眼看过去，变量“x”压根就不在“foo”中存在； 然而，正如我们下面要看到的——仅仅只是“乍一眼看过去“而已。我们看到“foo”上下文的活跃对象中只包含一个属性——“y”

```js
fooContext.AO = {
  y: undefined //undefined - 在进入上下文时，20 在激活阶段
}
```
那么，“foo”函数到底是如何访问到变量“x”的呢？一个顺其自然的想法是：函数应当有访问更高层上下文变量对象的权限。 而事实也恰是如此，就是通过函数的内部属性[[Scope]]来实现这一机制的。

>[[Scope]]是一个包含了所有上层变量对象的分层链，它属于当前函数上下文，并在函数创建的时候，保存在函数中。

这里要注意的很重要的一点是：[[Scope]]是在函数创建的时候保存起来的——静态的（不变的），只有一次并且一直都存在——直到函数销毁。 比方说，哪怕函数永远都不能被调用到，[[Scope]]属性也已经保存在函数对象上了。

另外要注意的一点是：[[Scope]]与Scope(作用域链)是不同的，前者是函数的属性，后者是上下文的属性。 以上述例子来说，“foo”函数的[[Scope]]如下所示：

```js
foo.[[Scope]] = [
  globalContext.Vo // global
]
```
之后，有了函数调用，就会进入函数上下文，这个时候会创建活跃对象并且this的值和Scope（作用域链）都会确定。下面来详细介绍下。

#### 函数激活

正如在“定义”这节提到的，在进入上下文，AO/VO创建之后，上下文的Scope属性（作用域链，用于变量查询）会定义为如下所示：

```js
Scope = AO|VO + [[SCope]]
```
这里要注意的时活动对象是Scope数组的第一个元素。添加到作用域链的最前面

```js
Scope = [AO].concat([[SCope]])
```
此特性对处理标识符非常重要。

>处理标识符其实就是一个确定变量（或者函数声明）属于作用域链中哪个变量对象的过程。

此算法返回的总是一个引用类型的值，其base属性就是对应的变量对象（或者如果变量不存在的时候则返回null），其property name属性的名字就是要查询的标识符。 要详细了解引用类型可以参看[this](/docs/this.md)。

标识符处理过程包括了对应的变量名的属性查询，比如：在作用域链中会进行一系列的变量对象的检测，从作用域链的最底层上下文一直到最上层上下文。

因此，在查询过程中上下文中的局部变量相比较上层上下文的变量会优先被查询到，换句话说，如果两个相同名字的变量存在于不同的上下文中时，处于底层上下文的变量会优先被找到。

```js
var x = 10;
function foo() {
  var y = 20;
  function bar() {
    var z = 30;
    alert(x +  y + z);
  }
  bar();
}
foo(); // 60
```
针对上述代码，对应了如下的变量/活跃对象，函数的[[Scope]]属性以及上下文的作用域链：

全局上下文的变量对象如下所示：
```js
globalContext.VO === Global = {
  x:10,
  foo:
}
```
在foo函数创建的时候，其[[Scope]]属性如下所示：

```js
foo.[[Scope]] = [
  globalContext.VO
]
```
在foo函数机会的时候(进入上下文时)，foo函数上下文的活跃对象如下所示
```js
fooContext.AO = {
  y: 20,
  bar:
}
```
同时，foo函数上下文的作用域链如下所示：

```js
fooContext.Scope = fooContext.AO + foo.[[Scope]]
fooContext.Scope = [
  fooContext.AO,
  globalContext.VO
]
```
在内部bar函数创建的时候，其[[Scope]]属性如下所示
```js
bar.[[SCope]] = [
  fooContext.AO,
  globalContext.VO
]
```
同时，“bar”函数上下文的作用域链如下所示：

```js
barContext.Scope = barContext.AO +  bar.[[Scope]]

barContext.Scope = [
  barContext.AO,
  fooContext.AO,
  globalContext.VO
]
```
如下是'x','y','z'标识符的查询过程：

```js
'x'
barContext.AO //not found
gooContext.AO //not found
globalContext.VO //found 10
'y'
barContext.AO //not found
fooContext.AO //found
'z'
barCOntext.AO //found
```
#### 作用域的特性

#### 闭包
在ECMAScript中，闭包和函数的[[Scope]]属性息息相关。正如此前介绍的，[[Scope]]是在函数创建的时候就保存在函数对象上了，并且直到函数销毁的时候才消失。 事实上，闭包就是函数代码和其[[Scope]]属性的组合。因此，[[Scope]]包含了函数创建所在的词法环境（上层变量对象）。 上层上下文中的变量，可以在函数激活的时候，通过变量对象的词法链（函数创建的时候就保存起来了）查询到。
```js
var x = 10;
function foo() {
  alert(x);
}
(function () {
  var x = 20;
  foo(); // 10, but not 20
})();
```
我们看到变量“x”是在“foo”函数的[[Scope]]中找到的。对于变量查询而言，词法链是在函数创建的时候就定义的，而不是在使用的调用的动态链（这个时候，变量“x”才会是20。
```js
function foo() {
 
  var x = 10;
  var y = 20;
 
  return function () {
    alert([x, y]);
  };
 
}
var x = 30;
var bar = foo(); // anonymous function is returned
bar(); // [10, 20]
```
上述例子再一次证明了处理标识符的时候，词法作用域链是在函数创建的时候定义的——变量“x”的值是10，而不是30。 并且，上述例子清楚的展示了函数（上述例子中指的是函数“foo”返回的匿名函数）的[[Scope]]属性，即使在创建该函数的上下文结束的时候依然存在。

#### 通过Function构造器创建的函数的[[Scope]]属性

在前面的例子中，我们看到函数在创建的时候就拥有了[[Scope]]属性，并且通过该属性可以获取所有上层上下文中的变量。 然而，这里有个例外，就是当函数通过Function构造器创建的时候。

```js
var x = 10
function foo() {
  var y = 20;
  function barFD() { // FunctionDeclaration
    alert(x);
    alert(y);
  }
  var barFE = function () { // FunctionExpression
    alert(x);
    alert(y);
  };
  var barFn = Function('alert(x); alert(y);');
  barFD(); // 10, 20
  barFE(); // 10, 20
  barFn(); // 10, "y" is not defined
}
foo();
```
上述例子中，函数“barFn”就是通过Function构造器来创建的，这个时候变量“y”就无法访问到了。 但这并不意味着函数“barFn”就没有内部的[[Scope]]属性了（否则它连变量“x”都无法访问到了）。 问题就在于当函数通过Function构造器来创建的时候，其[[Scope]]属性永远都只包含全局对象。 哪怕在上层上下文中（非全局上下文）创建一个闭包都是无济于事的。

#### 二维作用域链查询

在作用域链查询的时候还有很重要的一点：变量对象的原型（如果有的话）也是需要考虑的——因为原型是ECMAScript天生的特性：如果属性在对象中没有找到，那么会继续通过原型链进行查询。 比方说如下这些二维链：(1)在作用域链的链接上，(2)在每个作用域链接上——深入到原型链的链接上。如果在原型链（Object.prototype）上定义了属性就能观察到效果了

```js
function foo(){
  alert(x)
}
Object.prototype.x = 10
foo() //10
```
活跃对象是没有原型这一说的。通过如下例子可以看出：
```js
function foo() {
  var x = 20;
  function bar() {
    alert(x);
  }
  bar();
}
Object.prototype.x = 10;
foo(); // 20
```
试想下，如果“bar”函数的活跃对象有原型的话，属性“x”则应当在Object.prototype中找到，因为它在AO中根本不存在。 然而，上述第一个例子中，在标识符处理阶段遍历了整个作用域链，到了全局对象（部分实现是这样的），该对象继承自Object.prototype，因此，最终变量“x”的值就变成了10。

#### 全局和eval上下文的作用域链

全局上下文的作用域链中只包含全局对象。“eval”代码类型的上下文和调用上下文（calling context）有相同的作用域链。
```js
globalContext.Scope = [
  Global
]
evalContext.Scope === callingContext.Scope
```

#### 执行代码阶段对作用域的影响

ECMAScript中，在运行时，执行代码阶段有两种语句可以修改作用域链——with语句和catch从句。在标识符查询阶段，这两者都会被添加到作用域链的最前面。 比如，当有with或者catch的时候，作用域链就会被修改如下形式：

```js
Scope = withObject|catchObject + AO |VO + [[Scope]]

```
如下例子中，with语句添加了foo对象，使得它的属性可以不需要前缀直接访问

```js
var foo = { x: 10, y: 20 }
with(foo) {
  alert(x) // 10
  alert(y) // 20
}
```
对应的作用域链修改为如下所示：
```js
Scope = foo + AO|VO + [[Scope]]
```

```js
var x = 10, y = 10;
with ({x: 20}) {
  var x = 30, y = 30;
  alert(x); // 30
  alert(y); // 30
}
 
alert(x); // 10
alert(y); // 30
```
发生了什么？怎么最外层的“y”变成了30？ 在进入上下文的时候，“x”和“y”标识符已经添加到了变量对象。之后，到了执行代码阶段，发生了如下的改动：
- x=10, y=10
- 对象{x: 20}添加到了作用域链的最前面
- 在with中遇到了var语句，当然了，这个时候什么也不会发生。因为早在进入上下文阶段所有的变量都已经解析过了并且添加到了对应的变量对象上了。
- 这里修改了“x”的值，原本“x”是在第二步的时候添加的对象{x: 20}（该对象被添加到了作用域链的最前面）中的“x”，现在变成了30。
- 同样的，“y”的值也修改了，由原本的10变成了30
- 之后，在with语句结束之后，其特殊对象从作用域链中移除（修改过的“x”——30，也随之移除），作用域链又恢复到了with语句前的状态。
- 正如在最后两个alert中看到的，“x”的值恢复到了原先的10，而“y”的值因为在with语句的时候被修改过了，因此变为了30。

同样的，catch从句（可以访问参数异常）会创建一个只包含一个属性（异常参数名）的新对象。如下所示：
```js
try{
  ...
} catch(e) {
  calert(e)
}
```
作用域链修改为如下所示：
```js
var catchObject = {
  ex: 
};
Scope = catchObject + AO|VO + [[Scope]]
```