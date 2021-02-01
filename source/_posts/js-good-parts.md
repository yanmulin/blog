---
title: 《Javascript语言精粹》 小结
date: "2019/7/25"
categories:
- unclassified
---

### Javascript的语言精粹

作者对Javascript的评价是

> 在JavaScript中，美丽的、优雅的、富有表现力的语言特性就像一堆珍珠和鱼目混杂在一起。

对于Javascript这门语言，作者建议只应用一个精华的子集。

作者总结了Javascript三个语言精粹：

1. 函数是顶级对象

2. 基于原型继承的动态对象

3. 对象字面量和数组字面量


### 函数是顶级对象

在Javascript中，函数也是对象。即意味着函数也是一个键值对的集合，而且拥有一个连到原型对象的隐藏连接。函数对象连接到Function.prototype，而Function.prototype连接到Object.prototype。

### 基于原型继承的动态对象

原型模式在概念上比继承简单的多：新对象继承旧对象的属性。对象是无类别的，我们可以通过普通赋值给任何对象添加一个新成员属性。

当检索一个对象的属性时，会沿着原型链一直寻找，寻找到最后还是找不到则返回undefined。每个对象都连接到一个原型对象，所有字面量对象都连接到Object.prototype。

对于原型，要注意

1. 原型连接只在检索时用到，无法更新原型链上的属性；
2. for-in会枚举到原型链上的属性，可以通过`hasOwnProperty()`方法过滤掉这些属性。

### 对象字面量和数组字面量

在Javascript中创建对象字面量和数组字面量非常方便。

对象字面量的属性名可以用或不用引号括住。若属性名含有特殊字符，如‘ ’，‘-’，则必须用引号括住。

```javascript
var empty_object = {};

var stooge = {
"first-name": "Jerome",
"last-name": "Howard"
};
var departure = {
IATA: "SYD",
time: "2004-09-22 14:55",
city: "Sydney"
},”
```

检索时，非引号的属性用点运算符访问；有引号的属性用方括号运算符访问。

```javascript
stooge["first-name"]
flight.departure.IATA
```

Javascript中数组本质仍是对象。不过数组对象的键是数字/数字字符串。允许索引是不连续的，中间的元素用`undefined`填充。元素允许是不同类型的对象。数组字面量的用法与Python类似，


### Helper 函数

书中用到一系列Helper函数。

```javascript
Function.prototype.method = function (name, func) {
    this.prototype[name] = func;
    return this;
};
Function.method('inherits', function (Parent) {
    this.prototype = new Parent();
    return this;
});
Object.create = function (o) {
    var F = function () {};
    F.prototype = o;
    return new F();
};
```

## 函数

### 调用

函数调用时有两个附加参数：`this`和`arguments`。`this`的值根据调用模式不同而动态绑定到不同的对象。

1. 方法调用模式

当函数是对象的一个属性，`this`绑定到该对象上。

2. 函数调用模式

当函数不是对象的一个属性，`this`绑定到全局对象。

有时我们希望`this`能绑定到调用环境的`this`上，我们约定用`that`变量来表示外部的`this`。

```javascript
myObject.double = function () {
    var that = this;
    function helper() {
        return that.value + that.value;
    }
    return helper;
}
```

3. 构造器调用模式

当使用new来调用一个函数时，会自动创建一个对象。该对象的原型连接到该函数，`this`绑定到该对象上。

构造器在伪类模式中应用。而我们不建议使用伪类模式，因为存在更好的替代模式。

4. Apply调用模式

`apply`是函数对象拥有的方法，它有两个参数，第一个参数是`this`的绑定对象，第二个参数是`arguments`。

### 作用域

Javascript不存在C语言的块级作用域，但是有函数作用域。函数内部定义的变量，函数内部任何位置都可以访问，对函数外部则不可见。

```javascript
var a = 1;                          // 可访问a
var foo1 = function () {
    var b = 2;                      // 可访问a, b, d
    var foo2 = function () {
        var c = 3;                  // 可访问a, b, c, d
        // ...
    };
    foo2();
    var d = 4;
    foo2();
};
foo1();
```

### 闭包

函数可以访问它在被创建时所处的上下文环境。只要内部函数需要，变量会一直保留。

1. 利用闭包构造私有属性

```javascript
var myObject = function () {
    int value = 1;
    return {
        increment: function () {
            value += 1;
        },
        get_value: function () {
            return value;
        }
    };
}();
```

外部无法访问`myObject`的`value`属性，只能通过`increment`和`get_value`方法。

2. 一个糟糕的例子

```javascript
var add_the_handlers = function (nodes) {
    var i;
    for (i=0;i<nodes.length;i++) {
        nodes[i].onclick = function (e) {
            alert(i);
        }
    }
}
```

`onclick`触发时，`i`的取值都是`nodes.length`。因为Javascript的闭包都是引用捕获。

正确示范

```javascript
var add_the_handlers = function (nodes) {
    var helper = function (i) {
        return function (e) {
            alert(i);
        }
    }
    var i;
    for (i=0;i<nodes.length;i++) {
        nodes[i].onclick = helper(i);
    }
}
```


### 级联（链式模式）

对象方法都返回`this`，可以实现在一个语句中链式的调用方法。

### 模块模式

模块模式是隐藏了状态和实现的对象或函数。模块一般是利用闭包封装私有变量，由一个**立即调用的匿名函数**实现，返回一个对象/函数。

### Curry化

Curry化的意思是改造函数的接口，返回一个新函数。

```javascript
Function.method('curry', function (args) {
    var slice = Array.prototype.slice;
    var args = slice.appley(arguments);
    var that = this;
    return function () {
        that.apply(null, args.concat(slice.apply(arguments)));
    };
});
```

注：arguments不是Array对象，没有`concat`方法，所以用`slice`转换成Array对象。

### 备忘录模式（装饰模式）

从顶至下递归容易重复计算（例：斐波那次数列），一个解决方案是利用备忘录模式。

```javascript
var memoizer = function (memo, formula) {
    var recur = function (n) {
        var result = memo[n];
        if (typeof result !== 'number') {
            result = formula(recur, n);
            memo[n] = result;
        }
    };
    return recur;
};
```

即formula函数的装饰器。


## 原型

### 伪类模式

用伪类模式定义一个类和实例。

```javascript
var Mamal = function (name) {
    this.name = name;
};
Mamal.prototype.get_name = function () {
    return this.name;
};
Mamal.prototype.says = function () {
    return this.saying || '';
};

var myMammal = new Mammal('Herb the Mammal');
var name = myMammal.name;
```

`new`运算符创建一个对象，该对象的原型连接到函数，调用构造器函数（`this`绑定到该对象上）。

用伪类模式实现继承。

```javascript
var Cat = function (name) {
    this.name = name;           // 父类的属性要重复的在子类中定义
    this.saying = 'meow';       // 子类增加的属性
}
Cat.prototype = new Mammal();
```

### 伪类模式的改进：差异化继承

```javascript
var myCat = Object.create(Mammal);
myCat.name = 'kitty';
myCat.saying = 'meow';
```

创建一个Mammal对象后，在myCat上动态的修改属性。

注：`Object.create`和`new`两种创建方法区别在于`Object.create`只连接原型，而不会调用构造器函数。

### 函数化模式

函数化模式的一般实现:

```javascript
var constructor = function (spec, my) {
    var that //, 私有属性
    my = my || {}
    
    // 将保护属性添加到my
    
    that = 新对象
    
    // 给that添加访问私有属性的方法
    
    return that
};
```

例：

```javascript
var Mammal = function (spec) {
    var that, name = spec.name || 'mammal'
    that = {}
    that.get_name = function () {
        return that.name;
    };
    that.say = function () { 
        return that.saying || '';
    };
    return that;
};
var Cat = function (spec) {
    var that, saying = spec.saying || 'meow';
    that = Mammal(spec);
    that.pur = function () { ... }
    return that;
};
```

## 附

书里第二章用铁路图(railroad diagram)的方式来介绍Javascript的语法。铁路图对应着巴科斯范式，但更直观。我觉得用这种方式来来学习一门语言的语法非常方便。

附上“铁路图”的维基百科：[https://en.wikipedia.org/wiki/Syntax_diagram](https://en.wikipedia.org/wiki/Syntax_diagram)

