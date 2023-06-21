---
layout: post
title:  "Angular State Management Best Practices"
date:   2023-05-06
published: true
comments: true
categories: [Angular, State management, Best Practices, ObservableState, Angular Signals]
cover: assets/angular-state-management-best-practices.jpg
description: "This article explains best practices on how to deal with state in Angular"
---

Best Practices are mostly a matter of personal preference and can be countered by people with different opinions.
That being said, the Best Practices in this article are based on a decade of working with Single Page applications and managing state.
I have been on more than 100 Angular projects in the last 7 years and I have seen tons of different approaches and learned
the mindset of hundreds and hundreds of different professionals.

I have done flux, redux, state models, @ngrx/store, Akita, Ngxs, BehaviorSubjects, [ObservableState](https://github.com/simplifiedcourses/observable-state){:target="_blank"}, Signals...
I have seen a lot of things and this article is about how I reason about state.

## What is state management?

This might seem like a trivial question but there are a lot of different opinions on this topic.
The first question that we want to ask ourselves is: **"What is state?"**.
**"State is a value or object that is being kept in memory"**:
- A property on a component that holds a value is considered state.
- A property on a service that holds a value is considered state.
- The value of an `@Input()` property is considered state.
- Some value being kept in some kind of state management framework store like @ngrx is considered state.
- Some value that is provided on ngModule level or route level is considered state.

```typescript
@Component({...}
export class HelloCompontent {
    // This message property is state
    public message = 'hello';
}
```

Now what is state management? We will talk about state management when one or more of these statements is true:
- When we want to update state
- When we want to add state
- When we want to invalidate state
- When we want to share state between components
- When we want to share state between features
- When we want to persist state in the db, localstorage etc

### First misconception

A lot of developers think that state management means: **sharing state globally across the application**.
That is not true, and by reasoning about state like that, people tend to manage to much state or manage it wrong.

State management is as simple as the following example:

```typescript
@Component({...}
export class HelloCompontent {
    // This message property is state
    public message = 'hello';

    // This is state management
    public update(newMessage: string): void {
        this.message = newMessage;
    }
}
```

### Second misconception

Whether you love creating actions for everything that triggers effects, that trigger other effects that in their turn will trigger other effects is a decision that every team has to make for themselves. But let's not pretend that is state management. It's orchestration and communication in most cases.

## Best Practice Number 1: KISS (Keep it simple, stupid)

When this principle is valid for most solutions in software development (or even in life), it's very important **to keep things simple** when it comes to state management. Angular is moving from RxJS-driven state management toward Signal-driven state management.
This sends a very clear message: The Angular core team wants to make it simpler to do state management.

At Simplified Courses, we avoid over-engineered solutions where we are dependent on big third-party frameworks. We avoid boilerplate code that infects every layer of our frontend architecture. We stay as close to Angular, and Typescript and only manage state that we need to manage.

## Best practices number 2: Don't manage state you shouldn't manage

This boils down to this: **Don't put everything into one store**. If you do:
- You will need to jump through hoops to get that state.
- You will wind up with a lot of boilerplate code and maintenance cost.
- You will need to invalidate that state at a given time
- You will glue all the feature modules of your application together
- You will wind up with a complex store design


Take this example, for instance, it can't be simpler than this right?!

```typescript
@Component({
    ...
    template: `
    <user-form [user]="user$|async"></user-form>
    `
})
export class UserComponent {
    ...
    public user$ = this.userId$.pipe(
        switchMap(id =>this.userService.getById(id))
    );

}
```
This simple example:
- fetches data for us every time the `userId$` changes
- Subscribes automatically for us
- Fetches the data automatically for us on subscription
- Unsubscribes automatically for us
- Nothing to invalidate

By putting this into a @ngrx store for instance we would introduce the following complexity:
- An action, actionType, effects, selectors, reducers have to be written.
- We don't have access to the response of `getById()` directly, we have to add that in the store.
- We don't have access to errors directly, we have to add those in the store too if we want to access them.
- We have to invalidate the user when the component gets destroyed, so in the `ngOnDestroy()` lifecycle hook we have to send a destroy action, etc...

The general rule is: **If your state doesn't need to be shared, don't put it in a store**.
Now this isn't an article about state management frameworks, so let's summarize the Best Practice as **"Don't unnecessarily persist state in other places if it doesn't have an added value"**

## Best Practice number 3: Keep your state as low as possible

What we mean by that is if you need your state in the lowest child component, and only there... **Keep it there!**
- If you need to share a piece of state between 2 sibling components, keep the state in the next parent component.
- If you need to share a piece of state between smart components, keep that state provided in the feature.
- If you really need a piece of state to be shared between different lazy loaded modules, in that case, we are talking about global state and that is the highest level. Always try to keep it as low as possible. Future you will thank you later!

Angular has this great dependency injection feature where we can provide an injectable on all different kinds of levels.
Take this example for instance:

```typescript
@Component({
    ...
    // provided right on this ChatboxComponent
    providers: [ChatboxState]
})
export class ChatboxComponent {
    private readonly state = inject(ChatboxState);
}
```

Now the `ChatboxState` is sandboxed for the `ChatboxComponent` and will share the life cycle of that component as well.
That means that when `ChatboxComponent` gets destroyed, the instance of `ChatboxState` will also get destroyed.
You can even implement the `ngOnDestroy()` lifecycle hook on the `ChatboxState` as well:

```typescript
export class ChatboxState implements OnDestroy {
    // gets called when the item that provides this class gets destroyed
    public ngOnDestroy(): void {
    }
}
```

The lower you keep the state, the easier it will become for us to manage that state, and the more we get for free.
In this scenario, state invalidation is happening for free for instance.

## Best Practice number 4: Keep state synchronous, always provide snapshots

A healthy way of thinking about state is that state should always have a value.
That means that we don't always want to subscribe to the value, but we want to get a snapshot of that value.


Take this bad example:

```typescript
public saveUser(): void {
    this.user$
        .pipe(
            take(1),
            withLatestFrom(this.courses$),
            takeUntil(this.destroy$)
        )
        .subscribe(({user, courses}) => {
            this.userService.update(user, courses).subscribe({...})
        })
}
```

Because `user$` and `courses$` are both observables we can't just get values from them. We have to subscribe to them in order to retrieve those values.
If our state is synchronous this piece of code becomes a lot cleaner:

```typescript
public saveUser(): void {
    const {user, courses} = this.state.snapshot;
    this.userService.update(user, courses).subscribe({...})
}
```

That piece of code is using [ObservableState](https://github.com/simplifiedcourses/observable-state){:target="_blank"} but that is also
what Signals are all about. With Signals it would look like this:

```typescript
public saveUser(): void {
    const {user, courses} = this.state();
    this.userService.update(user, courses).subscribe({...})
}
```

## Best Practice number 5: Always provide initial values

When working with state, It's a healthy way to always think about initial values:
Avoid the value `null` if you can, in some cases you will need it but instantiating an array with `[]` will make your code more robust.
You will wind up with less **"Cannot read properties of null (reading 'forEach')"** and it makes reasoning about state easier.
Your typing of your state would also become easier since you wouldn't have to add `<your-type>|null` after every state type.
This for instance get can quite annoying:

```typescript
type UserState = {
    firstName: string|null,
    lastName: string|null
    // etc
}
```

## Best Practice number 6: Immutable data structures

Immutable data structures aren't necessarily used because of performance optimizations.
The reason why we use it is because these structures are predictable.
We could optimize Change Detection, we can leverage RxJS operators and we can create clean
unidirectional dataflows

## Wrapping up

That's a wrap! Whether we use state management frameworks or custom implementations, try to keep it simple,
keep the state as low as possible, ensure you have initial values and snapshots and don't manage state you shouldn't be managing

I hope you liked it. If you have feedback, please leave a comment

If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training){:target="_blank"} and [Angular Coaching](https://www.simplified.courses/angular-coaching){:target="_blank"}