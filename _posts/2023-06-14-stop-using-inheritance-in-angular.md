---
layout: post
title:  "Stop using in inheritance in Angular"
date:   2023-06-10
published: true
comments: true
categories: [Angular, Architecture, Best Practices]
description: "This article explains why we should stay away from inheritance in Angular"
---

At Simplified Courses, we tend to do **a lot of Angular code-reviews** for companies that want to make sure they are still on the right track. One of the principles that often makes their codebase unmaintainable and unnecessarily complex is the principle of **inheritance**.
This article explains why we should stay away from inheritance in general and what the dangers are when using inheritance.
After that, we will cover an elegant alternative.

## Why do people use inheritance in the first place?

At school, we are taught to think **DRY**. DRY stands for **"Don't repeat yourself"**. Don't repeat any code so make sure if you have 2 classes that need the same code, that this code isn't duplicated. In some cases, **KISS** (**"Keep it Simple Stupid"**) is more important than DRY and we still duplicate pieces of code in favor of flexibility. In this article we will focus on DRY and how to keep our codebase DRY. The first thing that they taught us at school is to use inheritance, where we extract the redundant logic that we need to share in a base class. Take this code for instance:

```typescript
class Car {
    public start(): void {...}
    public stop(): void {...}
    ...
}

class Bike {
    public start(): void {...}
    public stop(): void {...}
    ...
}
```

We can see that both of these classes have a `start()` method and `stop()` method implemented. This functionality is redundant so it is not DRY.
We can fix this by creating a `Vehicle` base class that implements this:

```typescript
class Vehicle {
    public start(): void {...}
    public stop(): void {...}
}
class Car extends Vehicle{
    ...
}

class Bike extends Vehicle {
    ...
}
```

As we can see, the `Car` class and the `Bike` class, both inherit from that base class and fix the duplicate logic issue that we had before. The nice thing is, our logic is in one central place and by inheriting from that base class we get this logic for free.
However, **we consider this a bad practice** and we will explain why.

## Why we should not use inheritance in Angular (or any technology for that matter)?

Inheritance is a bad practice in general because we tend to inherit more than we want.
When we inherit from a base class we inherit everything from that base class so inheritance:
- Breaks single responsibility pattern
- Is hard to test
- Breaks separation of concerns
- Becomes very complex in no-time

Let's say that we have a class `Foo` that needs the following functionalities:
- Logging
- Data fetching
- Caching in `localStorage`

We could inherit the class `Foo` from `Base` like this and every instance of `Foo` would have the three functionalities

```typescript
export class Base {
    log() {...}
    fetch() {...}
    cache() {...}
}
export class Foo extends Base {
    // acccess to logging, data fetching and caching
}
```

Imagine we now have a class `Bar` that needs logging, data fetching but it does not need caching. However it also needs access to the authentication, so now this class needs:
- Logging
- Data fetching
- Authentication

The logical thing would be to add the authentication part to the `Base` class as well and let `Bar` inherit like this:

```typescript
export class Base {
    log() {...}
    fetch() {...}
    cache() {...}
    authenticate() {...}
}
export class Foo extends Base {
    // acccess to logging, data fetching and caching
}
export class Bar extends Base {
    // acccess to logging, data fetching and authentication
}
```

Our `Base` class is becoming dirtier because now it has 4 functionalities that are unrelated and `Foo` has access to authentication when it doesn't need to, and `Bar` has access to caching when it doesn't need to. This clearly breaks the single responsibility pattern and when our application grows, the `Base` class would grow in functionality and no one would even know what is inside that `Base` class. By pushing this principle further we could wind up with base classes inheriting from other base classes and so on.

Wouldn't this snippet make more sense?

```typescript
export class Logger {
    log() {...}
}
export class DataAccess {
    fetch() {...}
}
export class Cacher {
    cache() {...}
}
export class Authenticator {
    authenticate() {...}
}
export class Foo  {
    // acccess to logging, data fetching and caching
    private logger = inject(Logger);
    private dataAccess = inject(LoDataAccessgger);
    private cacher = inject(Cacher);
}
export class Bar {
    // acccess to logging, data fetching and authentication
    private logger = inject(Logger);
    private dataAccess = inject(LoDataAccessgger);
    private authenticator = inject(Authenticator);
}
```

This is easy to mock out, breathes single responsibility and is easy to read. No magic happening in a base class here

## Composition

Composition is the principle where we group pieces of code in other classes and inject them into the class rather than letting the class inherit from a base class. In Angular, we can leverage Dependency Injection for that. A simple solution to the previous example could be having an `Engine` class that can be injected inside both `Car` and `Bike`.

```typescript
class Engine {
    public start(): void {...}
    public stop(): void {...}
}
class Car {
    private engine = inject(Engine);
    ...
}

class Bike {
    private engine = inject(Engine);
    ...
}
```

** There is one reason, and one reason only why you could use inheritance in Angular projects:
When all the child classes need all the functionality of your base class **


## Dependency injection in Angular