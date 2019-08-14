### let 是什么?

相信大家对`let`并不陌生,毕竟`ES2019`了都,当我们问`let`是什么其实是在说它到底有什么用?与`var`有什么不同,

相信这是很多基础的面试题的部分,大多数人的答案是下面这些

- 不可重复生命
- 存在暂时性死区
- 可生成块级作用域
- 最后一个可能有些人会注意到,在全局环境下`let`声明的变量不会像`var`一样挂载到`window`上(毕竟现在的开发模式很少会直接这么做)

这个问题就是在一个偶然的环境下我无意发现的,也就是为啥名字为啥会带上两个不是太强相关的东西,因为这一次解开了我两个疑惑

### 进入正题

首先我们来看一段代码

```js
var windowNum = 0;
function wrapper() {
  let wrapperNum = 1;
  function test() {
    let testNum = 2;
    return wrapperNum + testNum;
  }
  return test;
}
let currentTest = wrapper();
console.dir(currentTest);
```

这段代码集齐了我们标题上的四个元素,`let`与`var`,闭包与作用域链,那么让我们来思考一个简单的问题,运行`currentTest`会返回什么?相信都能知道是 3,因为`wrapperNum`是 1,`testNum`2,加一起等于三,然后因为`test`形成了闭包,所以它能够读取到跟他处于同一作用域下的`wrapperNum`,所以可以得到正确的结果,

那么说到这里,我们再来说一下'`wrapperNum`是如何拿到的?'我们说闭包,作用域都是建立在我们看到的情况下,但是 V8 不会用眼睛看着这段代码去执行,所以就必然需要一个机制来让`currentTest`被推入执行环境的时候可以获取到它需要的值,我们回过头来看一下`console.dir(currentTest);`

![](\Screenshot_1.png)

我们看一下`[[Scope]]`这个内部属性?同时对比一下我们思考中的作用域链是否相同,`[[Scope]]`属性就是对作用域链的具体表现,当`currentTest`被执行,其内部会使用`wrapperNum`和`testNum`,引擎会按顺序对`[[Scope]]`内部的对象进行遍历,进行匹配,一旦获取到同名变量就进行下一步,这也就是为什么同名变量会使用离得相对近的,因为其属性在`[[Scope]]`中相对靠前,那么这又有`let`和`var`有什么关系呢? 我们分析一下`[[Scope]]`中的值,第一个`Closure`字面就可以看出是闭包的意思,当然也确实是闭包,第二个`Script`,我们暂时跳过,第三个`Global`可以看出是全局环境,也就是`window`,那么`Script`是什么?我们换一个简单的函数就好理解了.

```js
let foo = 1;
var foo1 = 2;
function fooTest() {}
console.dir(fooTest);
let fooTest1 = function() {};
console.dir(fooTest);
```

我们看一下这个的结果,

![](\Screenshot_3.png)

我们看到,通过`let`声明的变量都会被放置在`Script`中,而通过`var`声明的则不会,我们回头看一下`let可生成块级作用域`这个特性,那么`Script`是不是函数声明时的作用域呢? 我们来看第三个例子

```js
let testScript = 1;
function foo() {
  let testFunctionScope = 2;
  function test() {}
  console.dir(test);
}
foo();
```

结果如下,

![](\Screenshot_4.png)

通过这个例子,我们可以看出,`Script`是一个在`window`下的作用域,也就是说只有在`window`环境下通过`let`声明的变量才会被放入其中,而通过`var`声明的贼会被挂载到`window`上成为一个属性,那么不禁要尝试一下了,通过`let`与`var`去重复声明同一个变量是否可行呢?很可惜,是不行的,应该是引擎在声明是会检查两者,避免其发生重复.

tip:最后,尽信书不如无书,请各位试验之后做出判断,如有错误,请不吝指正
