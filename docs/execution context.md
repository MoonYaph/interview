### [执行上下文](http://dmitrysoshnikov.com/ecmascript/chapter-1-execution-contexts/)


#### 定义

每次当控制器转到ECMAScirpt可执行代码的时候，就会进入一个执行上下文。执行上下文(**EC**)是一个抽象概念，ECMA-262标准用这个概念同可执行代码(executable code)概念进行区分。
活动的执行上下文在逻辑上组成一个堆栈，堆栈底部永远是全局上下文(**global context**)，堆栈顶端是当前的执行上下文，堆栈在EC类型的变量(various kinds of EC)被推入或弹出的同时被修改。

#### 可执行代码

可执行代码的概念与抽象的执行上下文的概念是相对的。在某些时候，可执行代码与执行上下文是等价的。

```javascript
ECStack = []
```
每次进入函数(即使函数被递归调用或者作为构造函数)的时候或者内置的eval函数工作的时候，这个堆栈都会被推入。

#### 全局代码

这种类型的代码是在"程序"级处理的：例如加载外部的js文件或者本地的在<script></script>标签内的代码。全局代码不包括任何函数体内的代码
在初始化阶段，ECStack是这样的
```javascript
ECStack = [
  globalContext
]
```
#### 函数代码

当进入函数代码(所有类型的函数),ECStack被推入新元素。要注意的是，具体的函数代码不包括内部函数(inner functions)代码。如下所示，我们使函数自己调自己的方式递归一次：

```javascript
(function foo(bar){
  if(bar){
    return 
  }
  foo(true)
})()
```
那么，ECStack以如下方式被改变：

```javascript
// first activation of foo
ECStack = [
  <foo>functionContext
  globalContext
]
// recursive activation of foo
ECStack = [
  <foo> functionContext - recursively
  <foo> functionContext
  globalContext
]
```
每次返回存在的当前执行上下文和ECStack弹出相应的执行上下文的时候，栈指针会自动移动位置，这是一个典型的堆栈实现释放。一个被抛出但是没有被截获得异常，同样存在一个或多个执行上下文。当相关段代码执行完成以后，直到整个应用程序结束，ECStack都只包括全局上下文(global context)

#### Eval代码

它有一个概念，调用上下文(calling context), 这是一个当eval函数被调用的时候产生的上下文。eval(变量或函数声明)活动会影响调用上下文(calling context)。

 ```javascript
eval('var x = 10')
(function foo(){
  eval('var y = 20')
})()
alert(x) // 10
alert(y) // y is not defined
 ```

 ```javascript
ECStack = [
  globalContext
]
// eval('var x = 10')
ECStack = [
  evalContext,
  callingContext: globalContext
]

//eval exited context
ECStack.pop()

// foo function call
ECStack.push(<foo> functionContext)

// eval('y = 20')
ECStack.push(
  evalContext,
  callContext: <foo> functionContext
)
// return from eval
ECStack.pop()

// return from foo
ECStack.pop()
 ```

 在版本号1.7以上的SpiderMonkey(内置于Firefox,Thunderbird)的实现中，可以把调用上下文作为第二个参数传递给eval。那么，如果这个上下文存在，就有可能影响“私有”(类似以private关键字命名的变量一样)变量。


 ```javascript
function foo() {
  var x = 1;
  return function () {
    alert(x)
  }
}
var bar = foo()

bar() // 1

eval('x = 2', bar) // pass context, influence on interval var x

bar() // 2

 ```