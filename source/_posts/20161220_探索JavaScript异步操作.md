---
title: '探索JavaScript异步操作'
date: 2016-12-20 22:46:23
tags:
  - JavaScript
  - 技术
  - 编程
categories: 编程
thumbnail: //static.gogo.moe/files/2016/12/thumbnail.jpg
---

对一门语言认识的加深，是一个不断踩坑再从坑里爬出来的过程。
这次Gogo跳进了JavaScript异步操作与回调函数的大坑。
学习过程中，看到了这个问题是如何一步步被解决的，仿佛自身也经历了技术发展的过程。

### 万物初始之时

碰巧项目里有这样一个功能，我就以它为例吧。Gogo需要根据用户ID查询到一个用户，然后获取该用户的所有评论。
很简单的一个需求。包含两个异步操作，得到用户和得到用户评论。按Gogo原本的思路，它写出来应该像这样：
```JavaScript
var user = getUser(id);
var comments = getComments(user.comments);
```
其中的user看上去像是这样
```JavaScript
{
    name : "张三",
    id : 18,
    comments :  [74 ,102 ,157 ,244 ,245]//所有评论的id
}
```
看上去挺好的不是么，开始写getUser吧
```JavaScript
function getUser(id){
    $.getJSON('/user', {id: id}, function(user){
        //写到这里，Gogo一脸懵逼
    });
}
```

诶诶诶，不对啊，得到的user该如何return出去？

沉思了半分钟，从来没有写过异步的Gogo终于明白了，异步的写法还是与同步有很大区别的。
从此Gogo打开了新世界的<s>大门</s>大坑，一脚踩了进去。

### 阶段一: 回调函数

不就是要把异步获取的数据传出去嘛，很简单，调用的时候塞个回调函数进来不就好了。Gogo没敲几个键，一个新鲜的函数就出炉了。

```JavaScript
function getUser(id, callback){
    $.getJSON('/user', {id: id}, function(user){
        callback(user);
    });
}
```

Gogo低下头，盯了这个新鲜的面包一会儿，它被吓得变形了……
```JavaScript
function getUser(id, callback){
    $.getJSON('/user', {id: id}, callback);
}
```

多么完美是不是！我们用一行就完成了这个函数的主体部分。另一个函数也如法炮制

```JavaScript
function getComments(id, callback){
    $.getJSON('/comment', {id: id}, callback);
}
```

没用几行，两个函数都已经搞定了，剩下就是调用的问题了。
```JavaScript
getUser(id, function(user){
    getComments(user.comments, function(comments) {
        //do something ...
    })
})
```

哦天，这个调用可不简单。为了用它，Gogo居然使用了两个回调函数，还得嵌套！如果以后什么功能复杂点，回调次数多，有个六层七层什么的……

哇太可怕了，Gogo下出一身冷汗，这么丑的代码怎么看得下去！

回头看了一眼同步的写法
```JavaScript
var user = getUser(id);
var comments = getComments(user.comments);
```
多么漂亮，多么简洁，如果异步也能这么写那该多好。

结果真能这么写。

### 阶段二:Promise

据说Es6新特性中有个Promise，专门用来解决异步回调嵌套的问题。原本咱与Promise那哥们也不熟，这不，一遇到问题我就想起它了，果然遇到问题就得找人帮忙啊。

咱就规规矩矩去拜访了一下 [Promise](//es6.ruanyifeng.com/#docs/promise)

它的最大功能就是把嵌套的回调函数变成串行。
```JavaScript
new Promise(function(resolve, reject) {
    resolve(7);
}).then(function(data) {
    console.log(data); // 7
    return {value:data,name:'hello'};
}).then(function(data) {
    console.log(data); // {value:7,name:'hello'}
})
```

上面这个例子都是同步的，如果需要在then中进行一个异步操作该怎么办呢？别担心，Promise老兄早就帮你做好了。

```JavaScript
Promise.resolve(null).then(function() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            resolve(`yes,it's me`);
        }, 1000); // 1秒后才结束执行操作
    });
}).then(function(data) {
    console.log(data);//yes,it's me
})

//注
Promise.resolve(null)
//相当于
new Promise(function(resolve, reject) {
    resolve(null);
})
```

你可以在then中返回一个Promise对象，它会等待这个Promise对象执行结束后，再把这个Promise的结果传递给下一个then函数。
利用这个特性，咱们就能愉快地写出代码了。

```JavaScript
/**
* 封装getJSON函数，让他返回一个 Promise对象
*/
function getJSON(url, data) {
    return new Promise(function(resolve, reject) {
        $.getJSON(url, data, resolve);
    });
}

getJSON('/user', {id:id})
    .then(function(user) {
        return getJSON('/comment', {id:user.comments});
    })
    .then(function(comments) {
        //do something
    });

//用Es6新语法的简化版
getJSON('/user', {id})
    .then(user => getJSON('/comment', {id:user.comments}))
    .then(comments => {
        //do something
    });
```

如此一来，回调嵌套的问题就被顺利解决了。

然而再次审视这段代码，为什么Gogo感觉还是太复杂了？固然没有同步写法那么简单，甚至比阶段一的写法还要复杂。

真是艹了喵了(Gogo惯用语，相当于日了狗了)，说好的简单快捷高效的写法呢？丫的被你吃了？

### 阶段三:Generator 与 co

[generator](//es6.ruanyifeng.com/#docs/generator)函数出现在Es6中并没有引起Gogo很大的关注，因为Gogo早已在Python的py交易中接触了它的存在。然而Gogo做梦也没想到，它还能跟异步扯上一腿。

先看看generator函数到底是个啥。

```JavaScript
function* gen() {
    var a = 10,
        b = 5;
    yield a;
    yield b;
    var c = yield;
    return a + b + c;
}

var g = gen();
g.next();  //{value: 10, done: false}
g.next();  //{value: 5, done: false}
g.next();  //{value: undefined, done: false}
g.next(7); //{value: 22, done: true}
g.next();  //{value: undefined, done: true}
```
Gogo觉得这个例子已经可以比较好地说明generator的实质了。可以把他看做一个比较特殊的函数，普通的函数只能返回一次，而generator能返回多次，每次yield返回一个值。
每次调用next()函数，可以让generator继续往下走一步。
并且还可以通过next传递参数。

好吧讲了这么多Gogo还是一头雾水，这货到底有啥用呢，怎么跟异步扯上关系的？这就不得不提一下TJ大神写的一个co函数了。

用它来写我们的代码，看起来会像是这样:

```JavaScript
function getUser(id) {
    return new Promise(function(resolve, reject) {
        $.getJSON('/user', {id}, resolve);
    });
}
function getComments(id) {
    return new Promise(function(resolve, reject) {
        $.getJSON('/comment', {id}, resolve);
    });
}

co(function* () {
    var user = yield getUser(id);
    var comments = yield getComments(user.comments);
    // do something
});
```

核心是最后这几行的内容。这里运用了generator函数可以运行到一半后暂停的特性，co函数接受了generator yield返回出来的promise，等待promise运行结束后，再将promise的结果重新赛回generator里，这就有了我们上面那种写法。

co函数是一个十分巧妙的设计，通过它就能完美地用同步的方式写异步操作。想了解co的实现原理的话，网上已经有一大堆解析
[ES6入门 co模块](//es6.ruanyifeng.com/#docs/async#co模块)
[koa实战 co精讲](//book.apebook.org/minghe/koa-action/co/index.html)
以及 [co的源码](https://github.com/tj/co)

看到这一段代码，Gogo顿时感觉被一种强烈的幸福感充满。
这种写法除了中间多了一个yield关键字外，几乎跟同步的代码一致，写起来真是太爽了！

### 阶段四:Async

co函数已经解决了Gogo的几乎所有问题，但是貌似还没完。

Es7还有个叫[Async](//es6.ruanyifeng.com/#docs/async#async函数)的东西，也是用来解决异步操作的。

实际上，它与co十分相似
```JavaScript
async function f() {
    var user = await getUser(id);
    var comments = await getComments(user.comments);
    // do something
}
f();
```
仔细看，它不过是把co中的yield换成了await罢了。

Async被称为异步操作的终极解决方案，有了Async，
我们就可以在不需要引入co库的情况下，完成优美的异步操作。

可惜的是，发文时，绝大多数平台都没有实现这个Es7的特性。
想要真正使用它，只有再等半年或者使用Babel转码器了。

但可以预见的是，将来的异步操作必然是Async的天下。

### 后记

写这篇文章的时候，Gogo比较脱线，大概就是一直没吃药的那种状态，各个段落语言风格也都有些不同。

Gogo写这篇文章并不是想作为一篇教程，内容并不是十分详细，深度也远远不够。
真要说是什么的话，估计算是我的学习笔记吧。

看到一个问题困扰着人们，人们则绞尽脑汁的去解决，体会技术发展的历程，也是件有趣的事呢。
