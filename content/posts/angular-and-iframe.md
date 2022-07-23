---
title: "踏破铁鞋无觅处"
date: 2022-07-11T19:56:44+08:00
draft: false
tags: 
    - tech
---

不太会写 tech blog，所以这篇主要是写上周前端新手解决一个陌生问题的过程，目的纯粹是给自己做一个记录。由于过程十分艰辛，但最后找到的实现方法非常简单，所以就取了这么个题目。

# Problem

在做的网页应用中，有一个 iframe 在页面中，用来显示用户指定网址的网站界面。为了防止一些 CORS 安全问题，用的方法是先把用户指定网址网站的原界面爬下来，包括 html, javascript, css，然后把这些文件写入 iframe 的 source document 里。这里的爬虫方法被写成了一个 API 接口，并且原网页中的`<a>` tag 下的链接都换成了进入这个 API 接口的链接，这样点击内部链接的时候，也是通过爬虫获取的内容。这样技术上来说，iframe 界面里的东西和主界面里的东西并没有跨 domain。**现在的主要任务是，识别鼠标在 iframe 中的“右键”操作，并且把原始的右键菜单换成自定义的右键菜单**。

由于在前端页面上，iframe 约等于另一个 window，所以普通 Angular 的 `(click)` 和 `(contextmenu)` 完全不起作用，只有 `(mouseover)`、 `(mouseout)` 和 `window.blur()` 能被识别到。之前在 StackOverflow 上找到的识别 `mouseclick` 的方法是，先识别是否有 `mouseover` event，然后检测 `iframe` 的 `window.blur` event，如果用户点击了 iframe 内部的内容，那 iframe 的 window 就会被 blur。总结起来看就是，如果在 iframe 上已经发生了 `mouseover` event，并且 window 也被 blur了，那么就可以确定 `click`  event 发生了。可惜的是，这个办法不能区分是鼠标左键还是右键的点击。

# First Try - Add class on clicked element's DOM

最开始想了一个比较天真的办法。由于我是把爬下来的 html，javascript 等写入 iframe 的 source document 里，那我可以在写入前对原网站的 javascript 进行修改，直接在原网站的 javascript 里加入 `contenxtmenu` 的识别，识别后的 callback function 是对点击的 element 的 html 上增加一个 class，类似 `right-clicked`。在 Angular 里，我已经在 iframe 上识别 `click`，那我只需要在识别 `click` 的方法里，添加几行代码，看看鼠标下的 element 的 `classList` 里有没有 `right-clicked`，如果有，就说明这是个右键点击。

**看上去好像行得通，屁，都是表象**。实际操作起来，发现识别右键需要点击两下，估计一下是 Angular 当中的 `window.blur()` 识别，一下是原网站被我新加的 `contextmenu` 识别。只能另外想新办法。

# Second Try - Use JQuery

在谷歌上搜了一圈，发现很多答案都是利用 JQuery 来识别。于是死马当活马医，在 Angular 里装了 JQuery，确实成功了，但是后面 Angular 不能及时识别网站的变化了，很多之前写好的功能都变得很奇怪。最初以为是 `focus` 出现了问题，浏览器的 `focus` 没有从 iframe 转回到主界面，结果调试了半天也没什么作用。搜了大半天，排除了各种可能因素，感觉问题应该在 JQuery 上。确实在 Angular 中使用 JQuery 会让 Angular 离开 `zone`，而这个 `zone` 就是用来检测 Angular 应用中的各种变化的，具体可以参考 [Zones in Angular](https://blog.thoughtram.io/angular/2016/02/01/zones-in-angular-2.html)。最后的解决办法是，在需要回到主 Angular 应用界面的时候，加上 `zone.run(() => {})`。

**看上去好像问题都解决，屁，都是表象**。在 iframe 重新 load 内容以后（整个应用没有刷新），检测完全失效了，就算放在 Angular 的 `(onload)` 下，也完全没有效果。

# Third Try - Insert an empty div and change attribute value of that div

由于 JQuery 的方法还是有问题，加上 Angular 里用 JQuery 检测变化还是不太保险，于是打算放弃 JQuery 这条路。思考半天，还是打算沿用第一个方法的思路。在写入 iframe 的 source document 前，直接修改爬下来的网页的 html 和 javascript。修改的步骤是，首先在爬下来的 html 的最后加入一个空白的 `div`，并且赋予其 `id`。然后在原网页的 javascript 中添加 `contenxtmenu` 识别，callback function 是在新的空白 `div` 中添加属性，属性的值是点击当时的鼠标坐标（会在生成自定义右键菜单时用到）。在 Angular 这边，用 `MutationObserver` 检测这个空白 `div` 的变化。这个检测放在 Angular 的 `onload` method 下面。写完整个流程，大致测试了一下，好像没什么问题，重新 load iframe 也不会让检测失效，也避免了 Angular 直接离开 zone。

**很完美的样子，屁，都是表象**。当我点击了 iframe 内部界面的链接以后，检测失效了。由于 `MutationObserver` 是在 `onload` method 下面，而点击在 iframe 中 load 的网页中的内部链接，并不会让 iframe 整个重新 load。哈，另寻新路吧。 

# Fourth Try - window.postMessage

其实这个 `window.postMessage` 在解决之前的问题中也看到过，是解决主页面和 iframe 沟通的一个途径之一。功能很简单，就是可以像指定的 window 发送 message，然后被指定的 window 可以添加 `eventListener` 来检测是否收到新 message。但是很多教程，包括 MDN 官网，给的例子都是从主页面给 iframe 发送信息，并且好像都要先打开一个实体的 window，比如像 `var popup = window.open(/* popup details */);` 所以让我以为，在 iframe 里面打开应用主页面的实体 window，并给其发送信息是不可行的。但是由于实在想不到其他办法，又打算死马当活马医。

还是直接修改爬下来的网页的 javascript。将检测 `contenxtmenu` 的 callback function 改成 `window.parent.postMessage('mouse-cordinates', 'main-application-domain');`。在 Angular 的这边，就直接检测 window 的 `message` event。大致的代码是

```javascript
@HostListener('window:message', ['$event'])
onMessage(e: MessageEvent) {
    if (e.origin == config.LoginRedirectUri) {
        const data = e.data;
        if (Object.keys(data).includes('x')) {
            this.iframeRightClick.emit(data);
        }
    }
}
```
**这次是真的解决了～**

感觉绕了很远的路，也不知道自己是不是脑子一根筋，思路打结，总感觉应该有更简单明了的办法，算了，想不动了，躺。