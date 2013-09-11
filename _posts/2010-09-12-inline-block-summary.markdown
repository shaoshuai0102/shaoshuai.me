---
layout: post
title: "inline-block总结"
date: 2010-09-12 16:36
comments: true
categories: tech
---

前几天做东西的时候遇到由于hasLayout引起的bug，google的时候搜到这篇[IE7-/Win: inline-block and hasLayout](http://www.brunildo.org/test/InlineBlockLayout.html)，简单总结了下。

在现代浏览器中inline-block behavior都可以方便的通过`display:inline-block`来触发，但是（恨！），在IE 7-（IE7及以下版本）中，`display:inline-block`却未必能触发inline-block行为。

友情提示，以下说的是IE 7－

### 对于内联元素

比如`<span><a><textarea><input>`等：

*   `display:inline-block;`可以触发inline-block behavior

*   `zoom:1;height:0;`可以触发inline-block behavior

在这里a和b两种方法都可以使元素获得hasLayout属性，故此，对于内联元素只要使其获得hasLayout属性就可以触发inline-block behavior。

### 对于块级元素

比如`<div><h1><p><li><ul><ol><li>`等：

*   `display:inline-block;`不可以触发inline-block behavior

*   `zoom:1;height:0;`不可以触发inline-block behavior

*   `zoom:1;height:0;display:inline;`可以触发inline-block behavior

在这里可以看出通过添加hasLayout属性不可以使块级元素触发inline-block behavior，但是当加上display:inline之后就可以触发了。甚至还存在一种看起来恨诡异的方法：

*   在两个选择器中按顺序分别定义`display:inline-block;`和`display:inline;`可以触发inline-block behavior。比如（必须按照这个顺序才可以，也不可以写在同一个选择器中，即使同一个选择器也要写两次）

        div.test { display: inline-block; }
        div.test { display: inline; }

综上，在IE 7-中，inline-block的触发条件是：**拥有hasLayout属性的内联元素**。

也就是说对于内联元素，只要让其获得hasLayout属性便可以触发；对于块级元素来说，需要先将其转化为内联元素，再加上hasLayout属性，这样才可以触发inline-block behavior。

那么，我们写一个通用类，实现兼容所有浏览器（包括IE和非IE现代浏览器），且在内联元素和块级元素上都可以触发inline-block行为：

    .inlineBlock {
        display:inline-block;
        *display:inline;
        *zoom:1;
        *height:0;
    }

注意第1行和2行是不能调转顺序滴:-)

<hr/>

ps:最近好衰，记得某晚我用windows live writer写的这篇文章，但是writer的突然crash让我今天重新写了一遍……

