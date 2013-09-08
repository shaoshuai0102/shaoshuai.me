---
layout: post
title: "A flexible event model in JavaScript -《Ajax In Action》"
date: 2009-11-03 15:41
comments: true
categories: tech
---

MVC中Controller的作用是作为Model和View的中介使两者能够彼此分离，在GUI程序中，比如AJAX客户端，Controller层由很多事件回调函数组成。现代浏览器支持两种不同的事件模型：传统的经典事件模型和正在取代取代传统模型的W3C建议的新事件模型。

###一、经典javascript事件处理模型

经典事件处理模型的用法如下：

    MyDomElement.onclick = myEventHandler;     //注意此处不要写成myEventHandler()!
    function myEventHandler(){
       //some skillfully executed code here
    }

Common GUI event handler properties in the DOM

*   onmouseover - Triggered when the mouse first passes into an element’s region.
*   onmouseout - Triggered when the mouse passes out of an element’s region.
*   onmousemove - Triggered whenever the mouse moves while within an element’s region (i.e., frequently!).
*   onclick - Triggered when the mouse is clicked within an element’s region.
*   onkeypress - Triggered when a key is pressed while this element has input focus. Global key handlers can be attached to the document’s body.
*   onfocus - A visible element receives input focus.
*   onblur - A visible element loses input focus.

我们可以利用经典模型封装一个简单的Button类：

    function Button(value,domEl) { 
        this.domEl=domEl; 
        this.value=value; 
        this.domEl.buttonObj=this; 
        this.domEl.onclick=this.clickHandler; 
    }

    Button.prototype.clickHandler=function() { 
      var buttonObj=this.buttonObj; 
      var value=(buttonObj && buttonObj.value) ? buttonObj.value : "unknown value"; 
      alert(value); 
    }

这里利用了一个小hack，将Button的句柄保存在DomElement中，这样在事件处理函数中可以访问保存在Button中的value。错误的做法是：

    function Button(value,domEl) { 
        this.domEl=domEl; 
        this.value=value; 
        this.domEl.onclick=this.clickHandler;
    }

    Button.prototype.clickHandler=function() { 
        alert(this.value); 
    }

在上面的代码中，无法显示正确的结果，因为传递给事件处理函数clickHandler的上下文是DomElement的，而不是Button对象的。

###二、W3C事件模型

W3C事件模型支持将多个事件监听函数绑定到同一DOM元素上。但是这一模型面临严重的浏览器兼容性问题：

    Browsers           |   none-ie                 |        IE
    -----------------------------------------------------------------------
    Methods            |   addEventListener()      |        attachEvent()
                       |   removeEventListener()   |        detachEvent()
    -----------------------------------------------------------------------
    context object     |   DOM element             |        Window object

第一个兼容性问题比较好解决，第二个问题比较棘手：由于IE浏览器中传递给事件处理函数的上下文是Window对象，使得无法确定到底哪个DomElement触发了该事件！

###三、SOLUTION

鉴于以上两个事件模型的讨论，目前两种实现方法都无法很好的满足我们的需求。那么我们应该如何既解决浏览器兼容性问题，又能够实现多函数监听呢？《Ajax In Action》的作者建议不要使用新的W3C事件模型，提出了一个解决方案：经典事件模型+Observer设计模式！

    JavaScript语言: EventRouter

    var jsEvent=new Array(); 
    jsEvent.EventRouter=function(el,eventType){ 
        this.lsnrs=new Array(); 
        this.el=el; 
        el.eventRouter=this; 
        el[eventType]=jsEvent.EventRouter.callback; 
    } 
    jsEvent.EventRouter.prototype.addListener=function(lsnr){ 
        this.lsnrs.append(lsnr,true); 
    } 
    jsEvent.EventRouter.prototype.removeListener=function(lsnr){
        this.lsnrs.remove(lsnr); 
    } 
    jsEvent.EventRouter.prototype.notify=function(e){ 
        var lsnrs=this.lsnrs; 
        for(var i=0;i<lsnrs.length;i++){ 
            var lsnr=lsnrs[i]; 
            lsnr.call(this,e); 
        } 
    } 
    jsEvent.EventRouter.callback=function(event){ 
        var e=event || window.event; 
        var router=this.eventRouter; 
        router.notify(e) 
    }

*   hack1：

    在Javascript中，`””`和`[]`的作用是相等的，也就是说`el.onmouseover`跟`el[‘onmouseover’]`是等价的。

*   hack2:

    在回调函数callback中，当事件发生时需要通知EventRouter对象进行处理（通知所有的事件处理函数），然而回调函数的上下文对象是DOM元素，需要做一下转换。本例采取的方法是在生成EventRouter对象时将EventRouter对象的句柄保存在Dom元素的一个属性eventRouter中（el.eventRouter=this;），然后在回调函数中取出（var router=this.eventRouter;）。

经过以上的封装，那么事件监听将变得十分优雅：

    JavaScript语言: EventRouter调用示例

    window.onload=function(){ 
        var mat=document.getElementById(‘mousemat’); 
        cursor=document.getElementById(‘cursor’); 
        var mouseRouter=new jsEvent.EventRouter(mat,"onmousemove"); 
        mouseRouter.addListener(eventHandler1); 
        mouseRouter.addListener(eventHandler2); 
    } 
    function eventHandler1(e){ 
        //Some code here 
    } 
    function eventHandler2(e){ 
        //Some code here 
    }

相信以上的代码会非常有用处，快把它加入你的复用宝库吧！

（以上内容大部分是我翻译自《Ajax in action》，作了部分注解，如有问题，请联系我。欢迎讨论。）
