---
layout: post
title: 性能优化 —— 重绘和重排、缓冲和事件委托
date: 2017-09-29
tags: [JavaScript,share]
description: [somethings about performance]
---

针对昨天面试的问题，接下来都会一步一步解决，今天先解决这个关于性能优化的问题。

## 重绘和重排

### 基本概念

当浏览器下载完页面中的所有component之后，就会产生两个内部的数据结构。

- <p>DOM树</p> 表示页面结构
- <p>渲染树</p>表示DOM节点如何显示

DOM树中的每一个需要显示的节点在渲染树中都是会有对应的节点。渲染树中的节点被称为帧或者是“盒”，简单理解就是我们平常所说的盒模型。一点DOM和渲染树构建完成，浏览器就会开始显示页面元素。

动DOM的变化影响了元素的几何属性（宽或高），增加了文字导致行数增加的时候，浏览器需要重新计算元素的几何属性，同样其他元素的几何属性和位置都会因此受到影响。浏览器会使渲染树中受到影响的部分失效，并重新构造渲染树，这个过程称为“重排（reflow）”，完成之后，浏览器会重新绘制受影响部分到屏幕当中，这个过程称为“重绘（repaint）”。

这有东西要注意，不一定是所有的DOM变化都会影响几何属性，例如：改变一个元素的背景色并不会影响它的宽和高，在这种情况下只会发生一次重绘，并不会发生重排，因为页面的布局并没有发生变化。重排重绘的操作都是代价高昂的，回导致web应用UI反应迟钝，应该要尽量减少这类操作。

### 重排发生时间

之前一直没总结过这方面的知识，现在讲一讲。

- 添加或者删除可见的DOM元素
- 元素位置发生改变。
- 元素尺寸改变（padding，margin，width，height，border等）
- 内容改变，例如，文本改变或者图片被另一个尺寸不同的图片代替。
- 页面渲染初始化
- 浏览器窗口尺寸改变。

根据改变范围的程度，渲染树中或大或小的对应部分也需要重新计算。有些改变会触发整个页面的重排：例如，滚动条出现。

### 渲染树变化的排队与刷新

由于每次重排都会产生计算消耗，大多数浏览器通过队列化修改并批量执行来优化重拍过程。但是有时候我们会不知不觉就会强制的刷新队列并要求计划任务立刻执行。获取布局的信息就会导致队列的刷新，例如以下方法：

- offsetTop offsetLeft offsetWidth offsetHight
- scrollTop scrollLeft scrollWidth scrollHeight 
- clientTop clientLeft clientWidth clientHeight
- getComputedStyle() (currentStyle in IE Browser)

这些属性和方法都是要求返回最新的布局信息，所以浏览器不得不执行渲染队列中的“待处理变化”，并且触发重排来返回最新结果。

所以，我们最好避免使用上面的属性，他们都会刷新渲染队列，即使你是在获取最近未发生改变的或者与最新改变无关的布局信息。

### 最小化重绘和重排

先来个例子：

    var el = document.getElementById("myDiv");
    el.style.borderLeft = "1px";
    el.style.borderRight = "2px"
    el.style.padding = "5px";

这个例子中，三个样式属性被改变，每一个改变都会影响元素的几何结构，最重要的是，会导致浏览器触发了三个重排。

#### 优化方式

1.合并所有改变，一次性改变

    var el = document.getElementById("myDiv");
    el.style.cssText = "border-left: 1px; border-right: 2px; padding: 5px;";

这个例子中代码修改了cssText属性并且覆盖了已存在的样式信息，因此如果想保留现有样式，可以把它附加在CSSText字符串后面。

    el.style.cssText += "; border-left: 1px";

2.使用类名修改样式

这种方法用于那些不依赖运行逻辑和计算的情况。改变css的class名称的方法更加清晰，易于维护。

    var el = document.getElementById("myDiv");
    el.className = "active";

## 批量修改DOM

当我们需要对DOM进行一系列的操作的时候，我们需要通过以下步骤来进行优化：

    first 使元素脱离文档流
    second 对其应有多重改变
    finally 把文档从新引入文档中

#### 脱离文档流

有以下方法使DOM脱离文档:

- 隐藏元素，应用修改，重新显示
- 使用文档碎片（document.fragment），在当前DOM之外构建一个子树，再把它拷贝到文档中。
- 将元素拷贝到一个脱离文档节点中，修改副本，完成后在替代原始元素。

其中推荐使用文档碎片形式，因为它产生的DOM遍历和重排最少。

    <ul id="mylist">
        <li><a href=hh.com""></a></li>
        <li><a href="dd.com"></a></li>
    </ul>
    //更新a标签地址
    var fragment = document.createDocumentFragment();
    var data = [{
            "name" : "steven",
            "url" : "leunggabou.com"
        },
        {
            "name" : "leung",
            "url" : "steven"
        }
    ]
    function appendDataToElement(appendToElement,data){
        var a,li;
        for(var i = 0, max = data.length; i < max; i++){
            a = document.createElement('a');
            a.href = data[i].url;
            a.appendChild(document.cretaTextNode(data[i].name));
            li = document.createElement('li');
            li.appendChild(a);
            appendToElement.appendChild('li');
        }
    }
    appendDataToElement(fragment,data);
    document.getElementById('mylist').appendchild(fragment);

这种使用文档碎片减少重排的方法，在文档之外创建并更新一个文档碎片，然后将它附加到原来的列表中。它是一个轻量级的document对象，它的设计初衷就是为了完成这类任务的更新和移动节点。上面的例子，只触发了一次重排，而且只访问了一次实时的DOM。

## 缓冲布局信息

上面讲过，当我们访问页面获取一些位置信息，offset，client，scroll等都会触发重排。看看一下代码：

    myElement.style.left = 1 + myElement.offsetLeft + 'px';
    myElement.style.top = 1 + myElement.offsetTop + 'px';
    if( myElement.offsetLeft >= 500){
        stopAnimation();
    }

可以发现，这个方法是低效的，因为每次元素的移动都会查看一次偏移量，导致重排。一个更好的方法就是，获取一次起始位置的值，然后将其赋给一个变量，比如：

    var = current = myElement.style.left = current + 'px';
    myElement.style.top = current + 'px';
    myElement.style.top = current + 'px';
    if(current >= 500){
        stopAnimation();
    }

## 事件委托

相信这个大家都很熟悉了，当页面存在大量的元素，而且每一个都要一次或动词绑定事件的时候，这种情况就会影响性能。没绑定一个事件都是有代价的，他会加重页面的负担，或者会增加执行的时间。需要访问和修改的DOM元素越多，程序就越慢。

这个时候，我们就可以使用事件委托： 利用冒泡，事件逐层冒泡并能被父级捕获。使用事件委托，只需绑定父级原素，就可以触发器子元素的事件了。

捕获--->到达--->冒泡

这个时候，要提一下的就是事件源对象，它可以发现触发这个事件的元素。

    var ul= document.getElementsByTagName('ul')[0];
	ul.onclick = function (e) {
		var event = e || window.event;
		var target = event.target || event.srcElement;
		console.log(target.innerText);
	}

这个就是典型的一百万个li问题。。。每点击一个li都可以获取对应的内容，绑定事件是在父级上面的。

那么关于这部分的内容就先讲到这，希望大家能有所收获。再见。












