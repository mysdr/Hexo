---
title: 【深入浅出jQuery】源码浅析2--奇技淫巧
date: 2016-03-20 20:49:34
keywords: jQuery源码分析, jQuery源码, jQuery框架, jQuery架构
tags: jQuery源码分析
---
最近一直在研读 jQuery 源码，初看源码一头雾水毫无头绪，真正静下心来细看写的真是精妙，让你感叹代码之美。

其结构明晰，高内聚、低耦合，兼具优秀的性能与便利的扩展性，在浏览器的兼容性（功能缺陷、渐进增强）优雅的处理能力以及 Ajax 等方面周到而强大的定制功能无不令人惊叹。<!-- more -->

另外，阅读源码让我接触到了大量底层的知识。对原生JS 、框架设计、代码优化有了全新的认识，接下来将会写一系列关于 jQuery 解析的文章。

我在 github 上关于 jQuery 源码的全文注解，感兴趣的可以围观一下。[jQuery v1.10.2 源码注解 ](https://github.com/chokcoco/jQuery-)。

系列第一篇：[【深入浅出jQuery】源码浅析--整体架构](http://www.cnblogs.com/coco1s/p/5261646.html)

本篇是系列第二篇，标题起得有点大，希望内容对得起这个标题，这篇文章主要总结一下在 jQuery 中一些十分讨巧的 coding 方式，将会由浅及深，可能会有一些基础，但是我希望全面一点，对看文章的人都有所帮助，源码我还一直在阅读，也会不断的更新本文。

即便你不想去阅读源码，看看下面的总结，我想对提高编程能力，转换思维方式都大有裨益，废话少说，进入正题。

## 短路表达式 与 多重短路表达式

短路表达式这个应该人所皆知了。在 jQuery 中，大量的使用了短路表达式与多重短路表达式。

短路表达式：作为"&amp;&amp;"和"||"操作符的操作数表达式，这些表达式在进行求值时，只要最终的结果已经可以确定是真或假，求值过程便告终止，这称之为短路求值。这是这两个操作符的一个重要属性。
```javascript
// ||短路表达式
var foo = a || b;
// 相当于
if(a){
    foo = a;
}else{
    foo = b;
}

// &amp;短路表达式
var bar = a &amp;&amp; b;
// 相当于
if(a){
    bar = b;
}else{
    bar = a;
}
```

当然，上面两个例子是短路表达式最简单是情况，多数情况下，jQuery 是这样使用它们的：
```javascript
// 选自 jQuery 源码中的 Sizzle 部分
function siblingCheck(a, b) {
    var cur = b &amp; a,
        diff = cur &amp;&amp; a.nodeType === 1 &amp;&amp; b.nodeType === 1 &amp;&amp;
        (~b.sourceIndex || MAX_NEGATIVE) -
        (~a.sourceIndex || MAX_NEGATIVE);

    // other code ...
}
```

嗯，可以看到，diff 的值经历了多重短路表达式配合一些全等判断才得出，这种代码很优雅，但是可读性下降了很多，使用的时候权衡一下，多重短路表达式和简单短路表达式其实一样，只需要先把后面的当成一个整体，依次推进，得出最终值。
```javascript
var a = 1, b = 0, c = 3;

var foo = a &amp; b &amp;&amp; c, // 0 ,相当于 a &amp;&amp; (b &amp;&amp; c)
  bar = a || b || c;  // 1
```

这里需要提出一些值得注意的点：

1、在 Javascript 的逻辑运算中，0、""、null、false、undefined、NaN 都会判定为 false ，而其他都为 true

2、因为 Javascript 的内置弱类型域 (weak-typing domain)，所以对严格的输入验证这一点不太在意，即便使用 &amp;&amp; 或者 || 运算符的运算数不是布尔值，仍然可以将它看作布尔运算。虽然如此，还是建议如下：
```javascript
if(foo){ ... }     //不够严谨
if(!!foo){ ... }   //更为严谨，!!可将其他类型的值转换为boolean类型
```

注重细节，JavaScript 既不弱也不低等，我们只是需要更努力一点工作以使我们的代码变得真正健壮。

## 预定义常用方法的入口

在 jQuery 的头几十行，有这么一段有趣的代码：
```javascript
(function(window, undefined) {
    var
        // 定义了一个对象变量，一个字符串变量，一个数组变量
        class2type = {},
        core_version = "1.10.2",
        core_deletedIds = [],

        // 保存了对象、字符串、数组的一些常用方法 concat push 等等...
        core_concat = core_deletedIds.concat,
        core_push = core_deletedIds.push,
        core_slice = core_deletedIds.slice,
        core_indexOf = core_deletedIds.indexOf,
        core_toString = class2type.toString,
        core_hasOwn = class2type.hasOwnProperty,
        core_trim = core_version.trim;

})(window);
```

不得不说，jQuery 在细节上做的真的很好，这里首先定义了一个对象变量、一个字符串变量、数组变量，要注意这 3 个变量本身在下文是有自己的用途的（可以看到，jQuery 作者惜字如金，真的是去压榨每一个变量的作用，使其作用最大化）。

其次，借用这三个变量，再定义些常用的核心方法，从上往下是数组的 concat、push 、slice 、indexOf 方法，对象的 toString 、hasOwnProperty 方法以及字符串的 trim 方法，core_xxxx 这几个变量事先存储好了这些常用方法的入口，如果下文行文当中需要调用这些方法，将会：
```javascript
jQuery.fn = jQuery.prototype = {
    // ...

    // 将 jQuery 对象转换成数组类型
    toArray: function() {
        // 调用数组的 slice 方法，使用预先定义好了的 core_slice ，节省查找内存地址时间，提高效率
        // 相当于 return Array.prototype.slice.call(this)
        return core_slice.call(this);
    }
}
```

可以看到，当需要使用这些预先定义好的方法，只需要借助 call 或者 apply（[戳我详解](http://www.cnblogs.com/coco1s/p/4833199.html)）进行调用。

那么 jQuery 为什么要这样做呢，我觉得：

+ 以数组对象的 concat 方法为例，如果不预先定义好 core_concat = core_deletedIds.concat 而是调用实例 arr 的方法 concat 时，首先需要辨别当前实例 arr 的类型是 Array，在内存空间中寻找 Array 的 concat 内存入口，把当前对象 arr 的指针和其他参数压入栈，跳转到 concat 地址开始执行，而当保存了 concat 方法的入口 core_concat 时，完全就可以省去前面两个步骤，从而提升一些性能；

+ 另外一点，借助 call 或者 apply 的方式调用，让一些类数组可以直接调用数组的方法。就如上面是示例，jQuery 对象是类数组类型，可以直接调用数组的 slice 方法转换为数组类型。又譬如，将参数 arguments 转换为数组类型：
```javascript
function test(a,b,c){
    // 将参数 arguments 转换为数组
    // 使之可以调用数组成员方法
    var arr = Array.prototype.slice.call(arguments);

    ...
}
```

## 钩子机制（hook）

在 jQuery 2.0.0 之前的版本，对兼容性做了大量的处理，正是这样才让广大开发人员能够忽略不同浏览器的不同特性的专注于业务本身的逻辑。而其中，钩子机制在浏览器兼容方面起了十分巨大的作用。

钩子是编程惯用的一种手法，用来解决一种或多种特殊情况的处理。

简单来说，钩子就是适配器原理，或者说是表驱动原理，我们预先定义了一些钩子，在正常的代码逻辑中使用钩子去适配一些特殊的属性，样式或事件，这样可以让我们少写很多 else if 语句。

如果还是很难懂，看一个简单的例子，举例说明 hook 到底如何使用：

现在考公务员，要么靠实力，要么靠关系，但领导肯定也不会弄的那么明显，一般都是暗箱操作，这个场景用钩子实现再合理不过了。
```javascript
// 如果不用钩子的情况
// 考生分数以及父亲名
function examinee(name, score, fatherName) {
    return {
        name: name,
        score: score,
        fatherName: fatherName
    };
}

// 审阅考生们
function judge(examinees) {
    var result = {};
    for (var i in examinees) {
        var curExaminee = examinees[i];
        var ret = curExaminee.score;
        // 判断是否有后门关系
        if (curExaminee.fatherName === 'xijingping') {
            ret += 1000;
        } else if (curExaminee.fatherName === 'ligang') {
            ret += 100;
        } else if (curExaminee.fatherName === 'pengdehuai') {
            ret += 50;
        }
        result[curExaminee.name] = ret;
    }
    return result;
}

var lihao = examinee("lihao", 10, 'ligang');
var xida = examinee('xida', 8, 'xijinping');
var peng = examinee('peng', 60, 'pengdehuai');
var liaoxiaofeng = examinee('liaoxiaofeng', 100, 'liaodaniu');

var result = judge([lihao, xida, peng, liaoxiaofeng]);

// 根据分数选取前三名
for (var name in result) {
    console.log("name:" + name);
    console.log("score:" + score);
}
```

可以看到，在中间审阅考生这个函数中，运用了很多 else if 来判断是否考生有后门关系，如果现在业务场景发生变化，又多了几名考生，那么 else if 势必越来越复杂，往后维护代码也将越来越麻烦，成本很大，那么这个时候如果使用钩子机制，该如何做呢？
```javascript
// relationHook 是个钩子函数，用于得到关系得分
var relationHook = {
    "xijinping": 1000,
    "ligang": 100,
    "pengdehuai": 50,
　　 // 新的考生只需要在钩子里添加关系分
}

// 考生分数以及父亲名
function examinee(name, score, fatherName) {
    return {
        name: name,
        score: score,
        fatherName: fatherName
    };
}

// 审阅考生们
function judge(examinees) {
    var result = {};
    for (var i in examinees) {
        var curExaminee = examinees[i];
        var ret = curExaminee.score;
        if (relationHook[curExaminee.fatherName] ) {
            ret += relationHook[curExaminee.fatherName] ;
        }
        result[curExaminee.name] = ret;
    }
    return result;
}

var lihao = examinee("lihao", 10, 'ligang');
var xida = examinee('xida', 8, 'xijinping');
var peng = examinee('peng', 60, 'pengdehuai');
var liaoxiaofeng = examinee('liaoxiaofeng', 100, 'liaodaniu');

var result = judge([lihao, xida, peng, liaoxiaofeng]);

// 根据分数选取前三名
for (var name in result) {
    console.log("name:" + name);
    console.log("score:" + score);
}
```

可以看到，使用钩子去处理特殊情况，可以让代码的逻辑更加清晰，省去大量的条件判断，上面的钩子机制的实现方式，采用的就是表驱动方式，就是我们事先预定好一张表（俗称打表），用这张表去适配特殊情况。

当然 jQuery 的 hook 是一种更为抽象的概念，在不同场景可以用不同方式实现。

看看 jQuery 里的表驱动 hook 实现，$.type 方法：
```javascript
(function(window, undefined) {
    var
        // 用于预存储一张类型表用于 hook
        class2type = {};

    // 原生的 typeof 方法并不能区分出一个变量它是 Array 、RegExp 等 object 类型，jQuery 为了扩展 typeof 的表达力，因此有了 $.type 方法
    // 针对一些特殊的对象（例如 null，Array，RegExp）也进行精准的类型判断
    // 运用了钩子机制，判断类型前，将常见类型打表，先存于一个 Hash 表 class2type 里边
    jQuery.each("Boolean Number String Function Array Date RegExp Object Error".split(" "), function(i, name) {
        class2type["[object " + name + "]"] = name.toLowerCase();
    });

    jQuery.extend({
        // 确定JavaScript 对象的类型
        // 这个方法的关键之处在于 class2type[core_toString.call(obj)]
        // 可以使得 typeof obj 为 "object" 类型的得到更进一步的精确判断
        type: function(obj) {

            if (obj == null) {
                return String(obj);
            }
            // 利用事先存好的 hash 表 class2type 作精准判断
            // 这里因为 hook 的存在，省去了大量的 else if 判断
            return typeof obj === "object" || typeof obj === "function" ?
                class2type[core_toString.call(obj)] || "object" :
                typeof obj;
        }
    })
})(window);
```

这里的 hook 只是 jQuery 大量使用钩子的冰山一角，在对 DOM 元素的操作一块，attr 、val 、prop 、css 方法大量运用了钩子，用于兼容 IE 系列下的一些怪异行为。在遇到钩子函数的时候，要结合具体情境具体分析，这些钩子相对于表驱动而言更加复杂，它们的结构大体如下，只要记住钩子的核心原则，保持代码整体逻辑的流畅性，在特殊的情境下去处理一些特殊的情况：
```javascript
var someHook = {
    get: function(elem) {
        // obtain and return a value
        return "something";
    },
    set: function(elem, value) {
        // do something with value
    }
}
```

从某种程度上讲，钩子是一系列被设计为以你自己的代码来处理自定义值的回调函数。有了钩子，你可以将差不多任何东西保持在可控范围内。

## 连贯接口

无论 jQuery 如今的流行趋势是否在下降，它用起来确实让人大呼过瘾，这很大程度归功于它的链式调用，接口的连贯性及易记性。很多人将连贯接口看成链式调用，这并不全面，我觉得连贯接口包含了链式调用且代表更多。而 jQuery 无疑是连贯接口的佼佼者。

##### 链式调用

链式调用的主要思想就是使代码尽可能流畅易读，从而可以更快地被理解。有了链式调用，我们可以将代码组织为类似语句的片段，增强可读性的同时减少干扰。（[链式调用的具体实现上一章有详细讲到](http://www.cnblogs.com/coco1s/p/5261646.html)）
```javascript
// 传统写法
var elem = document.getElementById("foobar");
elem.style.background = "red";
elem.style.color = "green";
elem.addEventListener('click', function(event) {
  alert("hello world!");
}, true);

// jQuery 写法
$('xxx')
    .css("background", "red")
    .css("color", "green")
    .on("click", function(event) {
    　　alert("hello world");
    });
```

##### 命令查询同体

这个上一章也讲过了，就是函数重载。正常而言，应该是命令查询分离（Command and Query Separation，CQS），是源于命令式编程的一个概念。那些改变对象的状态（内部的值）的函数称为命令，而那些检索值的函数称为查询。

原则上，查询函数返回数据，命令函数返回状态，各司其职。而 jQuery 将 getter 和 setter 方法压缩到单一方法中创建了一个连贯的接口，使得代码暴露更少的方法，但却以更少的代码实现同样的目标。

##### 参数映射及处理

jQuery 的接口连贯性还体现在了对参数的兼容处理上，方法如何接收数据比让它们具有可链性更为重要。

虽然方法的链式调用是非常普遍的，你可以很容易地在你的代码中实现，但是处理参数却不同，使用者可能传入各种奇怪的参数类型，而 jQuery 作者想的真的很周到，考虑了用户的多种使用场景，提供了多种对参数的处理。
```javascript
// 传入键值对
jQuery("#some-selector")
  .css("background", "red")
  .css("color", "white")
  .css("font-weight", "bold")
  .css("padding", 10);

// 传入 JSON 对象
jQuery("#some-selector").css({
  "background" : "red",
  "color" : "white",
  "font-weight" : "bold",
  "padding" : 10
});
```

jQuery 的 on() 方法可以注册事件处理器。和 CSS() 一样它也可以接收一组映射格式的事件，但更进一步地，它允许单一处理器可以被多个事件注册：
```javascript
// binding events by passing a map
jQuery("#some-selector").on({
  "click" : myClickHandler,
  "keyup" : myKeyupHandler,
  "change" : myChangeHandler
});

// binding a handler to multiple events:
jQuery("#some-selector").on("click keyup change", myEventHandler);
```

## 无 new 构造

怎么访问 jQuery 类原型上的属性与方法，怎么做到做到既能隔离作用域还能使用 jQuery 原型对象的作用域呢？重点在于这一句：
```javascript
// Give the init function the jQuery prototype for later instantiation
jQuery.fn.init.prototype = jQuery.fn;
```

这里的关键就是通过原型传递解决问题，这一块上一章也讲过了，看过可以跳过了，将文字搬过来。

嘿，回想一下使用 jQuery 的时候，实例化一个 jQuery 对象的方法：
```javascript
// 无 new 构造
$('#test').text('Test');

// 当然也可以使用 new
var test = new $('#test');
test.text('Test');
```

大部分人使用 jQuery 的时候都是使用第一种无 new 的构造方式，直接 $('') 进行构造，这也是 jQuery 十分便捷的一个地方。当我们使用第一种无 new 构造方式的时候，其本质就是相当于 new jQuery()，那么在 jQuery 内部是如何实现的呢？看看：
```javascript
(function(window, undefined) {
    var
    // ...
    jQuery = function(selector, context) {
        // The jQuery object is actually just the init constructor 'enhanced'
        // 看这里，实例化方法 jQuery() 实际上是调用了其拓展的原型方法 jQuery.fn.init
        return new jQuery.fn.init(selector, context, rootjQuery);
    },

    // jQuery.prototype 即是 jQuery 的原型，挂载在上面的方法，即可让所有生成的 jQuery 对象使用
    jQuery.fn = jQuery.prototype = {
        // 实例化化方法，这个方法可以称作 jQuery 对象构造器
        init: function(selector, context, rootjQuery) {
            // ...
        }
    }
    // 这一句很关键，也很绕
    // jQuery 没有使用 new 运算符将 jQuery 实例化，而是直接调用其函数
    // 要实现这样,那么 jQuery 就要看成一个类，且返回一个正确的实例
    // 且实例还要能正确访问 jQuery 类原型上的属性与方法
    // jQuery 的方式是通过原型传递解决问题，把 jQuery 的原型传递给jQuery.prototype.init.prototype
    // 所以通过这个方法生成的实例 this 所指向的仍然是 jQuery.fn，所以能正确访问 jQuery 类原型上的属性与方法
    jQuery.fn.init.prototype = jQuery.fn;

})(window);
```

大部分人初看 jQuery.fn.init.prototype = jQuery.fn 这一句都会被卡主，很是不解。但是这句真的算是 jQuery 的绝妙之处。理解这几句很重要，分点解析一下：

1. 首先要明确，使用 $('xxx') 这种实例化方式，其内部调用的是 return new jQuery.fn.init(selector, context, rootjQuery) 这一句话，也就是构造实例是交给了 jQuery.fn.init() 方法取完成。

2. 将 jQuery.fn.init 的 prototype 属性设置为 jQuery.fn，那么使用 new jQuery.fn.init() 生成的对象的原型对象就是 jQuery.fn ，所以挂载到 jQuery.fn 上面的函数就相当于挂载到 jQuery.fn.init() 生成的 jQuery 对象上，所有使用 new jQuery.fn.init() 生成的对象也能够访问到 jQuery.fn 上的所有原型方法。

3. 也就是实例化方法存在这么一个关系链  

*   jQuery.fn.init.prototype = jQuery.fn = jQuery.prototype ;
*   new jQuery.fn.init() 相当于 new jQuery() ;
*   jQuery() 返回的是 new jQuery.fn.init()，而 var obj = new jQuery()，所以这 2 者是相当的，所以我们可以无 new 实例化 jQuery 对象。

## setTimeout

写到这里，发现上文的主题有些飘忽，接近于写成了 如何写出更好的 Javascript 代码，下面介绍一些 jQuery 中我觉得很棒的小技巧。

熟悉 jQuery 的人都知道 DOM Ready 事件，传Javascript原生的 window.onload 事件是在页面所有的资源都加载完毕后触发的。

如果页面上有大图片等资源响应缓慢, 会导致 window.onload 事件迟迟无法触发，所以出现了DOM Ready 事件。此事件在 DOM 文档结构准备完毕后触发，即在资源加载前触发。

另外我们需要在 DOM 准备完毕后，再修改DOM结构，比如添加DOM元素等。而为了完美实现 DOM Ready 事件，兼容各浏览器及低版本IE（针对高级的浏览器，可以使用 DOMContentLoaded 事件，省时省力），在 jQuery.ready() 方法里，运用了 setTimeout() 方法的一个特性， 在 setTimeout 中触发的函数, 一定是在 DOM 准备完毕后触发。
```javascript
jQuery.extend({
    ready: function(wait) {
        // 如果需要等待，holdReady()的时候，把hold住的次数减1，如果还没到达0，说明还需要继续hold住，return掉
        // 如果不需要等待，判断是否已经Ready过了，如果已经ready过了，就不需要处理了。异步队列里边的done的回调都会执行了
        if (wait === true ? --jQuery.readyWait : jQuery.isReady) {
            return;
        }

        // 确定 body 存在
        if (!document.body) {
            // 如果 body 还不存在 ，DOMContentLoaded 未完成，此时
            // 将 jQuery.ready 放入定时器 setTimeout 中
            // 不带时间参数的 setTimeout(a) 相当于 setTimeout(a,0)
            // 但是这里并不是立即触发 jQuery.ready
            // 由于 javascript 的单线程的异步模式
            // setTimeout(jQuery.ready) 会等到重绘完成才执行代码，也就是 DOMContentLoaded 之后才执行 jQuery.ready
            // 所以这里有个小技巧：在 setTimeout 中触发的函数, 一定会在 DOM 准备完毕后触发
            return setTimeout(jQuery.ready);
        }

        // Remember that the DOM is ready
        // 记录 DOM ready 已经完成
        jQuery.isReady = true;

        // If a normal DOM Ready event fired, decrement, and wait if need be
        // wait 为 false 表示ready事情未触发过，否则 return
        if (wait !== true &amp; --jQuery.readyWait &gt; 0) {
            return;
        }

        // If there are functions bound, to execute
        // 调用异步队列，然后派发成功事件出去（最后使用done接收，把上下文切换成document，默认第一个参数是jQuery。
        readyList.resolveWith(document, [jQuery]);

        // Trigger any bound ready events
        // 最后jQuery还可以触发自己的ready事件
        // 例如：
        //    $(document).on('ready', fn2);
        //    $(document).ready(fn1);
        // 这里的fn1会先执行，自己的ready事件绑定的fn2回调后执行
        if (jQuery.fn.trigger) {
            jQuery(document).trigger("ready").off("ready");
        }
    }
})
```

暂且写这么多吧，技巧还有很多，诸如 $.Deferred() 异步队列的实现，jQuery 事件流机制等，篇幅较长，将会在以后慢慢详述。

原创文章，文笔有限，才疏学浅，文中若有不正之处，万望告知。

系列第一篇：[【深入浅出jQuery】源码浅析--整体架构](http://www.cnblogs.com/coco1s/p/5261646.html)

最后，我在 github 上关于 jQuery 源码的全文注解，感兴趣的可以围观一下，给颗星星。[jQuery v1.10.2 源码注解 ](https://github.com/chokcoco/jQuery-)。
