---
title: 浏览器DOM之Event
date: 2022-04-01 17:29:00
categories: 
  - DOM
tags:
  - 浏览器
  - DOM
  - 事件
---


> DOM即Document Object Model(文档对象模型)

## Event

Event接口表示在DOM中出现的事件。

一些事件是*由用户触发的*,例如鼠标键盘事件；其他事件常由API生成，例如动画已经完成运行的事件，视频已被暂停等等。
事件也可以通过脚本代码触发，例如调用元素的`HTMLElement.click()`方法，或者定义一些自定义事件，再使用`EventTarget.dispatchEvent()`方法将自定义事件派发往指定的目标。

