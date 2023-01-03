---
layout: post
title:  "Overview of RxJS APIs in Angular"
date:   2022-12-25
published: false
comments: true
categories: Angular, RxJS
cover: assets/running-outputs-outside-zonejs.jpg
description: "We will learn about all the places where Angular exposes RxJS Observables in their public APIs"

---

Angular is a reactive framework and uses RxJS to expose reactive APIs with observables. This article lists the most important places where Angular exposes these observables and how we could use them.


## Http interaction

Let's start with the most important module which is called `HttpClientModule`. This module is used to perform XHR calls and performing an XHR call is an asynchronous task.

```typescript
const obs$ = this.httpClient.get('https://you-are.awesome');
```

This piece of code creates an observable that won't do anything until it's subscribed to.
The nice thing is that:
- A subscription will trigger the producer functions which will perform the XHR call
- It will not do the actual call until we subscribe to `obs$`
- If we unsubscribe (E.g on `ngOnDestroy`), it will perform an `xhr.abort()` behind the scenes

If we want to intercept these calls to add a **jwt** token for instance we could use an interceptor.
An interceptor would also use an observable

activatedRoute
Router.events
formControl
formGroup
formvalidator
interceptors
guards
resolvers?
recognizer
cdk backdrop
httpclient
viewChildren

isStable
Output