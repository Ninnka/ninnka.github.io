---
title: JavaScript-执行上下文和执行栈
date: 2021-06-11 11:25:08
categories:
  - JavaScript
  - 执行栈
tags:
  - JavaScript
	- 执行栈
---

## 代码的执行

说到代码的执行，大家的第一印象应该是一行一行的代码，一个函数一个函数的在顺序执行，但是在怪异的JavaScript中也是如此吗，我们先来看几个案例：

### 案例1
```javascript
function foo() {
  console.log('foo');
}

foo();

function bar() {
  console.log('bar');
}

bar();
// 结果
// foo
// bar
```

### 案例2

```javascript
function foo() {
  console.log('foo');
}

foo();

function foo() {
  console.log('bar');
}

foo();
// 结果
// foo
// foo
```

### 案例3

```javascript
function foo() {
  console.log('foo');
}

foo();

var foo = function() {
  console.log('bar');
}

foo();
// 结果
// foo
// bar
```

我们先来看上述三个案例的结果，对于案例1的结果应该是毫无疑问的，但是案例2、3的却与常识不相符。

原因在于，js代码在执行之前，js引擎会对代码先做分析，分析过程中把代码划分以一段一段可以执行的代码，这一段一段的可执行代码我们统称为执行上下文。

## 执行上下文（Execution context，简称EC）

### 执行上下文类型

执行上下文一般分为3种
- 全局执行上下文：全局上下文在浏览器中指的是`window`对象，也是`this` `globalThis`
- 函数执行上下文：函数被调用的时候会被创建，调用函数时都会创建一个新的执行上下文。
- eval执行上下文：通过`eval`运行的代码

<!-- more -->

执行上下文有几个重要的属性：
- `变量对象-Variable Object`
- `作用域链-Scope Chain`
- `this`

这里先不一一展开

## 执行上下文栈（EC stack）

我们一般会把执行上下文栈称为调用栈，本质是一个堆栈的结构，具有`后进先出(LIFO)`的特性，用于存储执行上下文对象。

```javascript
// 以这个js文件为例
function foo1() {
  console.log('foo1');
}
function foo2() {
foo1();
  console.log('foo2');
}
foo2();
```

我们创建一个数组来模拟这个栈结构，初始化时这个栈是空的

```typescript
const stack: ExecutionContext[] = [];
```

执行JS代码时，会先创建一个全局执行上下文（globalExecutionContext）并Push到当前的执行栈中，在浏览器关闭（页面的js进程退出）前都会存在。

```javascript
stack.push(globalContext as ExecutionContext);
// [globalContext]
```

发生函数时，js引擎都会为该函数创建一个新的函数执行上下文（functionExcutionContext）并Push到当前执行栈的栈顶。

```javascript
// 先执行了 foo2 函数
stack.push(foo2Context as ExecutionContext);
// [globalContext, foo2Context]

// 在 foo2 中执行了 foo1函数
stack.push(foo1Context as ExecutionContext);
// [globalContext, foo2Context, foo1Context]
```

当栈顶函数运行完成后，其对应的函数执行上下文将会从执行栈中Pop出，上下文控制权将移到当前执行栈的前一个执行上下文。

```javascript
// foo1执行后退出
stack.pop();
// [globalContext, foo2Context]

// foo2 执行后退出
stack.pop();
// [globalContext]
```

我们可以用这样一幅图来简单描述

![调用栈](https://user-images.githubusercontent.com/9619419/115104429-e74fc500-9f8a-11eb-9d52-c6abc8fce85b.png)
