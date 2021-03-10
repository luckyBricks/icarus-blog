---
title: JavaScript修饰器的使用研究
date: 2021-01-27 15:16:04
categories:
    - CRM
---

在学习Salesforce的Lightning Web Components框架时，时常见到`修饰器`的使用。

修饰器`Decorator`是ES7的一个提案，目前还是实验性质的新功能。在VScode中可以依照如下方式关闭使用警告。

![image-20210204105100705](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/image-20210204105100705.png)

修饰器用来解决两个问题：

1. 不同类之间共享方法
2. **编译期**对类和方法的行为进行改变

使用时在类或方法的上面加一个@符，有点像Java里的注解，但含义不太一致。

## 例子1：修饰类

```javascript
@setProp
class User{}

function setProp(target){
    target.age = 30;
}
console.log(User.age);
```

这个例子的含义就是：使用`setProp`方法修饰`User`类，并为`User`类增加了一个`age`的属性。此时修饰方法`setProp`中的参数`target`代表`User`类本身。

## 例子2： 使用额外参数修饰类

```javascript
@setProp(20)
class User{}

function setProp(value){
    return function(target){
        target.age = value;
    }
}

console.log(User.age); //20
```

此时修饰方法`setProp`提供了一个途径，借助修饰器动态地改变`User`类静态属性`age`的值。

```javascript
function testable(target){
    target.prototype.isTestable = true;
}
@testable
class MyTestableClass{
}

let obj = new MyTestableClass();
obj.isTestable; //true
```

## 例子3：修饰方法

类的方法的修饰器函数的第一个参数`target`指向类的原型对象。这是因为类中的方法实际上是作用于实例对象上的

```javascript
//Example1
class Person{
    @readonly
    getName(){
        return `${this.first} ${this.last}`
    }
}
function readonly(target, name, descriptor){
      	// descriptor对象原来的值如下：
  		// {
  		//   value: specifiedFunction,
  		//   enumerable: false,
  		//   configurable: true,
  		//   writable: true
  		// }
    descriptor.writable = false;
    return descriptor;
}


//Example2
class Math{
    @log
    add(a, b){
        return a + b;
    }
    
    function log(target, name, descriptor){
        const oldValue = descriptor.value;
        descriptor.value = function(){
            console.log(`调用了函数${name} `, arguments);
            return oldValue.apply(this, arguments);
        };
        return descriptor;
    }
}
```

在上述例子2中，`@log`修饰器的作用就是在执行原始的操作之前，执行一次`console.log`，从而达到输出日志的目的。

**修饰器只能用于类和类的方法，不能用于函数，因为存在函数提升；类是不会提升的。**