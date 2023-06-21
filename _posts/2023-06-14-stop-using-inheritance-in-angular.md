---
layout: post
title:  "Stop using in inheritance in Angular"
date:   2023-06-20
published: false
comments: true
categories: [Angular, Architecture, Best Practices]
description: "This article explains why we should stay away from inheritance in Angular"
---

At Simplified Courses, we tend to do **a lot of Angular code-reviews** for companies that want to make sure they are still on the right track. One of the principles that often makes their codebase unmaintainable and unnecessarily complex is the principle of **inheritance**.
This article explains why we should stay away from inheritance in general and what the dangers are when using inheritance.
After that, we will cover an elegant alternative.

## Why do people use inheritance in the first place?

At school, we are taught to think **DRY**. DRY stands for **"Don't Repeat Yourself"**. This means we shouldn't repeat any code so we have to make sure if we have 2 classes that need the same code, that this code isn't duplicated. In some cases, **KISS** (**"Keep it Simple Stupid"**) is more important than DRY so we still duplicate pieces of code in favor of flexibility. In this article we will focus on DRY and how to keep our codebase DRY. The first thing that they taught us at school was to use inheritance, where we extracted the duplicated logic that we needed to share in a base class. Take this code for instance:

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

We can see that both of these classes have a `start()` method and `stop()` method implemented. This functionality is redundant (duplicated) so it is not DRY.
We can fix this by creating a `Vehicle` base class that implements the `start()` and `stop()` method:

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

Inheritance is a bad practice in general because the classes that inherit from base classes tend to inherit more than they need.
When we inherit from a base class we inherit everything from that base class so inheritance:
- Breaks single responsibility pattern
- Is hard to unit-test, hard to mock
- Breaks separation of concerns
- Can become very complex in no-time
- Hard to read, we have no clue what kind of logic our base class has

Let's say that we have a class `Foo` that needs the following functionalities:
- Logging
- Data fetching
- Caching

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

The logical thing would be to add the authentication part to the `Base` class as well and let `Bar` inherit from `Base` like this:

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

Our `Base` class is becoming dirtier because now it has 4 functionalities that are unrelated and `Foo` has access to `authenticate()` when it doesn't need to, while `Bar` has access to `cache()` when it doesn't need to. This clearly breaks the single responsibility pattern and when our application grows, the `Base` class would grow in functionality and no one would even know what is inside that `Base` class. By pushing this principle further we could wind up with base classes inheriting from other base classes and so on.

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

This is easy to mock out, breathes single responsibility and is easy to read. No magic happening in a base class here. This is clearly a better approach, since we both `Foo` and `Bar` only have access to functionalities they actually need, and it is very composable.

## Composition

Composition is the principle where we group pieces of related code in classes and inject instances of those classes into the class rather than letting the class inherit from a base class. In Angular, we can leverage Dependency Injection for that.

We should always use composition unless the following statement is true:
**There is one reason, and one reason only why we could use inheritance in Angular projects:
When all the child classes need all the functionality of our base class.**


## Dependency injection in Angular

To use the composition pattern in Angular, we can leverage Dependency Injection.
The idea behind dependency injection is that we register a dependency to our framework and when we ask for that dependency the framework gives us an instance of that dependency.
 
```typescript
// Create an injectable dependency
@Injectable()
export class UserService {
    ...
}

@Component({
    ...
    // Register that injectable dependency to the root component
    // so it becomes a singleton
    providers: [UserService]
})
export class AppComponent {...}

@Component({...})
export class UserListComponent {
    // Inject an instance of that injectable wherever we like
    private readonly userService = inject(UserService);
}
```

This means we can break down all the logic into different injectables and inject them/mock them wherever we want. This makes it easy to pick different pieces of functionality from wherever we see fit.

### What about components?

Having singleton-instances for every dependency might not always be a good idea. Sometimes we want new instances of an injectable every time a component is created.
Sometimes the lifecycle of an Angular injectable needs to be shared with the lifecycle of a component. 

### The power of instances per component

Let's say that we want to share **phonenumber functionality** between this `AddUserComponent` and an `EditUserComponent`:

```typescript
@Component({...})
export class AddUserComponent {
    public phoneNumbers: string[] = [];
    public addPhoneNumber(): void { ... }
    public removePhoneNumber(): void { ... }
    public updatePhoneNumber(): void { ... }
    ...
}
@Component({...})
export class EditUserComponent {
    public phoneNumbers: string[] = [];
    public addPhoneNumber(): void { ... }
    public removePhoneNumber(): void { ... }
    public updatePhoneNumber(): void { ... }
    ...
}
```

This is redundant logic and a singleton will not fulfill our needs since it has a piece of state called `phoneNumbers`. We could let them both extend from a base class but we have already seen the downsides of that appraoch. Instead let's create an injectable called `PhoneNumberState` that has all that functionality and inject it in both the `AddUserComponent` and `EditUserComponent`. By adding it to the `providers` property of both components we will create 2 instances that will live and die along with their connected components and we won't have to share state between them.

```typescript
// Create an injectable 
@Injectable()
export class PhoneNumberState {
    public phoneNumbers: string[] = [];
    public addPhoneNumber(): void { ... }
    public removePhoneNumber(): void { ... }
    public updatePhoneNumber(): void { ... }
}

@Component({
    ...
    // Provide it for the first time
    // (connected to AddUsercomponent)
    providers: [PhoneNumberState]
})
export class AddUserComponent {
    // Inject the instance connected to AddUserComponent
    private readonly phoneNumberState = inject(PhoneNumberState);
    ...
}

@Component({
    ...
     // Provide it for the first time
    // (connected to EditUserComponent)
    providers: [PhoneNumberState]
})
export class EditUserComponent {
    // Inject the instance connected to EditUserComponent
    private readonly phoneNumberState = inject(PhoneNumberState);
    ...
}
```
We now have used composition to avoid duplicated code and when the `AddUserComponent` gets destroyed the instance of `PhoneNumberState` connected to that instance will be destroyed, and when the instance of `EditUserComponent` gets destroyed, the instance of `PhoneNumberState` connected to that instance will be destroyed.

We can even implement the `ngOnDestroy()` lifecycle hook on `PhoneNumberState` if we like:

```typescript
@Injectable()
export class PhoneNumberState implements OnDestroy {
    public phoneNumbers: string[] = [];
    public addPhoneNumber(): void { ... }
    public removePhoneNumber(): void { ... }
    public updatePhoneNumber(): void { ... }

    public ngOnDestroy(): void {
        // Add some teardown logic here
    }
}

```

### Sharing state between child components

Sometimes we want to share state/logic with child components and communicating with `@Input()` properties and `@Output()` properties is not possible. Let's take a wizard component for instance that has 3 steps. This would result in:
- A `WizardComponent` as a parent component
- A `Step1Component` as a child component of `WizardComponent`
- A `Step3Component` as a child component of `WizardComponent`
- A `Step3Component` as a child component of `WizardComponent`

We would want to share state between the child components but we also want to destroy the injectable when the wizard is completed (for instance when we navigate away from that wizard). In that case we can just provide the `Wizard` on the `WizardComponent` but inject and consume it in all the child components:

```typescript
// Create an injectable with Wizard state and functionality
@Injectable()
export class Wizard {...}

@Component({
    ...
    // Provide it once on the parent component
    providers: [Wizard]
})
export class WizardComponent { ... }

@Component({...})
export class Step1Component {
    // Inject it in Step1Component
    private readonly wizard = inject(Wizard);
}
@Component({...})
export class Step1Component {
    // Inject it in Step3Component
    private readonly wizard = inject(Wizard);
}
@Component({...})
export class Step1Component {
    // Inject it in Step3Component
    private readonly wizard = inject(Wizard);
}
```

The `WizardComponent`, `Step1Component`, `Step2Component` and `Step3Component` all share the same instance of the `Wizard` injectable. Which is nice for reusing state and functionality and the moment we navigate away from the `WizardComponent` the state would automatically be destroyed.

If we don't want to destroy that state, we could provide it on the `AppComponent` or use `providedIn: 'root'`:

```typescript
@Injectable({ providedIn: 'root'})
export class Wizard {...}
```

## Summary

We learned that inheritance can be dangerous and it makes our codebase hard to scale.
Composition is better because it follows the single responsibility pattern, it is more readable, more testable and easier to understand.

Angular has a great Dependency Injection system that we can use to create singletons.
But we can also create instances of a dependency on all the different component levels which gives us a lot of flexibility.

We can provide a dependency:
- As a singleton: with `providedIn: 'root'`
- As a singleton: At the `providers` property of the root component
- Tied to a component: At the `providers` property of any component

It's important to remember that when providing a dependency on a component we can implement the `ngOnDestroy()` lifecycle hook.

If you want to test your knowledge on the topic: "Angular Dependency injection", check out this free [Angular Dependency Injection Quiz](https://www.simplified.courses/free-angular-dependency-injection-quiz){:target="_blank"}.


If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training){:target="_blank"} and [Angular Coaching](https://www.simplified.courses/angular-coaching){:target="_blank"}