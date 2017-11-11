#### 定义

this时执行上下文中的一个属性:

```js
activeExecuationContext = {
  VO: {...}, // variable object
  this: thisValue
}
```
this与上下文中可执行代码的类型直接相关。this的值在进入上下文时确定，并且在上下文运行代码期间不会改变this的值

#### this在全局代码中的值

在全局代码中，this始终是全局对象本身，这样就可能间接的引用到它了
```js
// explicit property definition of the global object
this.a = 10; //global.a = 10
alert(a) // 10

// implicit definition via assigning to unqualified identifier
b = 20;
alert(this.b) // 20

// also implicit via variable declaration because variable object of the gllobal context is the global object iteself
var c = 30
alert(this.c) // 30
```

#### this在函数代码中的值

在函数代码中使用this时，应用场景不同会导致很多问题。this值得首要特点是没有静态绑定到一个函数
正如我们上面曾提到的那样，this的值在进入上下文时确定，在函数代码中，this的值每一次(进入上下文时)可能完全不同。
不管怎样，在代码运行期间，this的值是不变的，也就是说，因为this不是一个变量，所以不可能为其分配一个新值。（相反，在Python编程语言中，它明确的定义为对象本身，在运行期间可以不断改变）
```js
var foo = {x: 10};
var bar = {
  x: 20,
  test: function () {
    alert(this === bar); // true
    alert(this.x); // 20
    this = foo; // error
    alert(this.x); // if there wasn't an error then 20, not 10
  }
};
// on entering the context this value is
// determined as "bar" object; why so - will
// be discussed below in detail
bar.test(); // true, 20
foo.test = bar.test;
// however here this value will now refer
// to "foo" – even though we're calling the same function
foo.test(); // false, 10
```
那么，在函数代码中，什么影响了this的值发生变化？有几个因素。

首先，在通常的函数调用中，this是由激活上下文代码的调用者来提供的，即调用函数的父上下文(parent context)。this取决于调用函数的方式

为了在任何情况下准确无误的确定this值，有必要理解和记住这重要的一点：正是调用函数的方式影响了调用的上下文中this的值，没有别的什么（我们可以在一些文章，甚至是在关于javascript的书籍中看到，它们声称：“this的值取决于函数如何定义，如果它是全局函数，this设置为全局对象，如果函数是一个对象的方法，this将总是指向这个对象。–这绝对不正确”）。继续我们的话题，可以看到，即使是正常的全局函数也会因为不同调用方式而激活，这些不同调用方式产生了this不同的值。

```js
function foo() {
  alert(this);
}
foo(); // global
alert(foo === foo.prototype.constructor); // true
// but with another form of the call expression
// of the same function, this value is different
foo.prototype.constructor(); // foo.prototype
```
有时可能将函数作为某些对象的一个方法来调用，此时this的值不会设置为这个对象。

```js
var foo = {
  bar: function() {
    alert(this)
    alert(this === foo)
  }
}
foo.bar() // foo, true
var exampleFunc = foo.bar
alert(exampleFunc === foo.bar) // true
// again with another form of the call expression of the same function, we have different this value
exampleFunc() // global, false
```
那么，到底调用函数的方式如何影响this的值？为了充分理解this的值是如何确定的，我们需要详细分析一个内部类型(internal type)——引用类型（Reference type）。

#### 引用类型

用伪代码可以把引用类型表示为拥有两个属性的对象——base（即拥有属性的那个对象），和base中的propertyName 。
```js
var valueOfReferenceType = {
  base: <base object>,
  propertyName: <property name>
}
```
引用类型的值仅存在于两种情况中：
- 当我们处理一个标识符时
- 或一个属性访问器

标示符的处理过程在作用域链中讨论， 在这里我们只需要知道，使用这种处理方式的返回值总是一个引用类型的值
标识符时变量名。函数名。函数参数名和全局对象中未是别的属性名。
```js
var foo = 10
function bar() {}
```
在操作的中间结果中，引用类型对应的值如下：

```js
var fooReference = {
  bae: global,
  propertyNmae: 'foo'
}
var baseReference = {
  base: global,
  propertyName: 'bar'
}
```
为了从引用类型中得到一个对象真正的值，在伪代码中可以用GetValue方法来表示：
```js
function GetValue(value) {
  if (Type(value) != Reference){
    return value
  }
  var  base = GetValue(value)
  if(base === null) {
    throw new ReferenceError;
  }
  return base.[[Get]][GetPropertyName(value)]
}
```
内部的[[Get]]方法返回对象属性真正的值，包括对象原型链中集成属性的分析
```js
GetValue(fooReference) // 10
GetValue(barReference) // functiuon object bar
```
那么，从最重要的意义上来说，引用类型的值与函数上下文中的this的值是如何关联起来的呢？这个关联的过程是这篇文章的核心。(The given moment is the main of this article.) 在一个函数上下文中确定this的值的通用规则如下：

```js
function foo() {
  alert(this)
}
foo() //global
```
我们看到在调用括号的左边是一个引用类型值(因为foo是一个标识符)：

```js
var fooReference = {
  base: global,
  propertyName: 'foo'
}
```
相应地，this也设置为引用类型的base对象，即全局对象。

同样，在访问属性访问器时：

```js
var foo = {
  bar: function () {
    return this
  }
}
foo.bar() // this
```
同样，我们拥有一个引用类型的值，其base是foo对象，在函数bar激活时将base设置给this
```js
var fooReference = {
  base: foo,
  propertyName: 'bar'
}
```
但是，如果用另一种方式激活相同的函数，this的值将不同

```js
var test = foo.bar
test() //global
```
因为test作为标识符，产生了其他引用类型的值，该值得base(全局对象)被设置为this的值

```js
var testReference = {
  base: global
  propertyName: 'test'
}
```
现在，我们可以很明确的说明，为什么用不同的形式激活同一个函数会产生不同的this，答案在于不同的引用类型（type Reference）的中间值。

```js
function foo(){
  alert(this)
}
foo() //global

var fooReference = {
  base: global,
  propertyName: 'foo'
}
alert(foo === foo.prototype.constructor) // true
// another form of the call expression
foo.prototype.constructor() //foo.proptype, because
var fooPrototypeConstructorRefernce = {
  base: foo.prototype,
  propertyName: 'constructor
}
```
另一个通过调用方式动态确定this的值得例子

```js
function foo() {
  alert(this.bar)
}
var x = {bar: 10}
var y = {bar: 20}
x.test = foo
y.test = foo
x.test() // 10
y.test() // 20
```
#### 函数调用和非引用类型

那么，正如我们已经指出，当调用括号的左边不是引用类型而是其它类型，this的值自动设置为null，实际最终this的值被隐式转换为全局对象。
让我们思考下面这种函数表达书：
```js
(function (){
  alert(this) //null ->global
})()
```
在这个例子中，我们有一个函数对象但不是引用类型的对象(因为它不是标识符，也不是属性访问器)，相应地，this的值被设为全局对象

```js
var foo = {
  bar: function() {
    alert(this)
  }
}
foo.bar() // Reference OK -> foo
(foo.bar)() // Reference OK -> foo
(foo.bar=foo.bar)() //global
(false||foo.bar)() //global
(foo.barm foo.bar)() //global
```
那么，为什么我们有一个属性访问器，它的中间值应该为引用类型的值，但是在某些调用中我们得到this的值不是base对象，而是global对象？

问题出现在后面的三个调用，在执行一定的操作运算之后，在调用括号的左边的值不再是引用类型。

- 第一个例子很明显———明显的引用类型，结果是，this为base对象，即foo。
- 在第二个例子中，分组操作符(译者注：这里的分组操作符就是指foo.bar外面的括号"()")没有实际意义，想想上面提到的，从引用类型中获得一个对象真正的值的方法，如[GetValue](/docs/variable-object.md)。相应的，在分组操作的返回值中———我们得到的仍是一个引用类型。这就是this的值为什么再次被设为base对象，即 foo。
- 第三个例子中，与分组操作符不同，复制操作符调用了[GetValue](/docs/variable-object.md)方法。返回的结果已经是函数对象(不是引用类型)，这意味着this被设为null，实际最终结果被设置未global对象。
- 最后两个例子也是一样，都好操作符和逻辑操作符(OR)调用了GetValue方法，相应地，我们失去了引用类型的值而得到了函数类型的值，所以this的值被设为global的值。

#### 引用类型和this为null

有一种情况，如果调用方式确定了引用类型的值，不管怎样，只要this的值被设置为null，其最终就会被隐式转换成global，当引用类型的base对象是激活对象时，就会导致这种情况。
下面的实例中，内部函数被父函数调用，此时我们就能够看到上面说的那种特殊情况。局部变量、内部函数、形式参数都储存在给定函数的激活对象中。
```js
function foo() {
  function bar() {
    alert(this) //global
  }
  bar() //the same as AO.bar()
}
```
激活对象总是作为this的值返回——null（即伪代码AO.bar()相当于null.bar()）。(译者注：不明白参考这里)这里我们再次回到上面描述的情况，this的值最终还是被设置为全局对象。

有一种情况除外：“在with语句中调用函数，且在with对象(即下面例子中的__withObject)中包含函数名属性时”。with语句将其对象添加在作用域链最前端，即在激活对象的前面。那么对应的，引用类型有值(通过标识符或属性访问器)，其base对象不再是激活对象，而是with语句的对象。顺便提一句，这种情况不仅跟内部函数相关，还跟全局函数相关，因为with对象比作用域链里的最前端的对象(全局对象或一个激活对象)还要靠前:

```js
var x = 10;
with ({
  foo: function () {
    alert(this.x);
  },
  x: 20
}) {
  foo(); // 20
}
// because
var  fooReference = {
  base: __withObject,
  propertyName: 'foo'
};
```
在catch语句的实际参数中的函数调用存在类似情况：在这种情况下，catch对象被添加到作用域的最前端，即在激活对象或全局对象的前面。但是，这个特定的行为被确认为是ECMA-262-3的一个bug，这个在新版的ECMA-262-5中修复了。修复后，在特定的激活对象中，this指向全局对象。而不是catch对象。

```js
try {
  throw function () {
    alert(this);
  };
} catch (e) {
  e(); // __catchObject - in ES3, global - fixed in ES5
}
// on idea
var eReference = {
  base: __catchObject,
  propertyName: 'e'
};
// but, as this is a bug
// then this value is forced to global
// null => global
  
var eReference = {
  base: global,
  propertyName: 'e'
};
```

同样的情况出现在命名函数（函数的更多细节[Functions](/docs/function.md)）的递归调用中。在函数的第一次调用中，base对象是父激活对象（或全局对象），在递归调用中，base对象应该是存储着函数表达式可选名称的特定对象。但是，在这种情况下，this的值也总是被设置为global。

```js
(function foo(bar){
  alert(this)
  !bar&& foo(1) //should be special object but always global
})() //global
```
#### this在作为构造器调用的函数中的值

函数作为构造器调用时。
```js
function A(){
  alert(this)
  this.x = 10
}
var a = new A()
alert(a.x) // 10
```
在这个例子中，new操作符调用“A”函数内部的**[[Construct]]**方法，接着，在对象创建后，调用其内部的**[[Call]]**方法，所有相同的函数“A”都将this的值设置为新创建的对象。

#### 手动设置一个函数调用的this

在Function.prototype中定义了两个方法允许手动设置函数调用时this的值,它们是.apply和.call方法（所有的函数都可以访问它们）。它们用接受的第一个参数作为this的值，this在调用的作用域中使用。这两个方法的区别不大，对于.apply，第二个参数必须是数组（或者是类似数组的对象，如arguments，相反，.call能接受任何参数。两个方法必须的参数都是第一个——this。

```js
var b = 10
function a(c){
  alert(this.b)
  alert(c)
}
a(20) //this===global this.b==10 c==20
a.call({b:20}, 30)
a.apply({b: 30}, [40]) //this ==={b:30} this.b==30 c==40
```