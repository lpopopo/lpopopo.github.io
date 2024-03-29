## 原型链
 #### `__proto__和prototype`
__proto__指向当前对象的原型，prototype是函数才具有的属性，默认情况下，new 一个函数创建出的对象，其原型都指向这个函数的prototype属性。
一般只有内置的构造函数才拥有prototype属性 ， 代表当前的对象的作用域 ， 在new 初始化创造出来的时候 , 如下图
![构造函数初始化](../img/原型链-1.PNG)
```
b.__proto__ 指向了 a.prototype (a的原型链) , 而且此时的b是没有prototype属性的 ， 因为它是一个实列。
a因为是一个函数 ，所以a.__proto 指向的应该是Function的原型链
```
![](../img/原型链-2.PNG.jpg)
但是呢 Function.prototype其实是一个Object , 所以 
![](../img/原型链-3.jpg)
然而`Object.prototype.__proto__ === null`
构造函数原型链中的构造函数等于自身
![](../img/原型链-4.jpg)

### 用途
其实原型链的设计 ， 因为在js中没有面向对象的概念 ， 它可以在一定的程度上进行面向对象中继承 ， 多态的概念。以及在es6中的语法糖class ， 其实在内部也是基于js的原型链来进行的。