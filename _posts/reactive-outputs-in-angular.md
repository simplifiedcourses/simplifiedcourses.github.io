---
layout: post
title:  "Reactive Outputs in Angular"
date:   2022-12-25
published: false
comments: true
categories: Angular, RxJS
cover: assets/running-outputs-outside-zonejs.jpg
description: "In this article we will cover how we can use reactive Outputs in Angular to create clear reactive flows"

---

This article is about creating reactive Outputs. RxJS is an important dependency of Angular and `@Output()`s are normally initialized with EventEmitters. Not only EventEmitters can be used to initialize `@Output()`s but any type of Observable can be initialized to `@Output()`s.
To continue with this article we will have to do some reverse-engineering of `@Output()`. Let's dig deep!

## Digging into the source code

In this article we digged into the source code: todo