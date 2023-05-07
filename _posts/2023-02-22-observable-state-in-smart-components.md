---
layout: post
title:  "Observable component state in Angular"
date:   2023-02-22
published: true
comments: true
categories: [Angular, RxJS, State management]
cover: assets/observable-component-state-in-angular.jpg
description: "We will dive deep into Observable state in Angular Smart components that we can use to create better reactive flows and local state management."

---

This article is a follow-up article of the previous articles (newest to oldest):
- [Observable state for ui components in Angular](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}
- [Reactive input state for Angular ViewModels](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"}
- [Reactive ViewModels for Ui components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}

## Quick recap

We started with creating reactive [ViewModels for UI components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}. We learned how to keep our templates clean and have a fully observable model that we
can feed to the template (a specific reactive model for the template of a specific component). ViewModels are derived from local properties on that component and its `@Input()` properties. If we wanted them to be reactive we had to create setters and BehaviorSubjects for all of those properties that we later combined into a ViewModel by using the `combineLatest` and `distinctUntilChanged` operators.
In the following example, we see all the boilerplate we need to create a ViewModel for the `@Input()` properties: `firstName`, `lastName` and the local state property `collapsed`.

```typescript
// Manual BehaviorSubjects for input properties
private readonly firstName$$ = new BehaviorSubject<string>('');
private readonly lastName$$ = new BehaviorSubject<string>('');

// Manual setters for input properties
@Input() public set firstName(v: string) {
    this.firstName$$.next(v);
}

@Input() public set lastName(v: string) {
    this.lastName$$.next(v);
}

// Manual BehaviorSubject for collapsed state
private readonly collapsed$$ = new BehaviorSubject<boolean>(true);

public toggleCollapse(): void {
    // Abusing BehaviorSubject to create reactive state
    this.collapsed$$.next(!this.collapsed$$.value); 
}

// Combining it all in a ViewModel
public readonly vm$ = combineLatest({
    firstName: this.firstName$$.pipe(distinctUntilChanged()), 
    lastName: this.lastName$$.pipe(distinctUntilChanged()), 
    collapsed: this.collapsed$$.pipe(distinctUntilChanged())
})
```

We have 3 BehaviorSubjects, 3 setters, `combineLatest` boilerplate and `distinctUntilChanged` boilerplate.
If the `firstName` and `lastName` `@Input()` properties are set at the same time, we get multiple emissions of `vm$` at the same time which is bad for performance.
There is also not a **single source of truth** to talk to here and we have to know RxJS pretty well to achieve this pattern.

The `toggleCollapse()` function also looks dirty: It feels like we should abstract away the BehaviorSubject.

The good thing is, the template looks clean: We can extract logic from the template into the ViewModel and we only have one async pipe with one subscription.

```html
<ng-container *ngIf="vm$|async as vm">
    <h3>Person details <button (click)="toggleCollapse()">Toggle</button></h3>
    <div *ngIf="!vm.collapsed">
        {{vm.firstname}} {{vm.lastName}}
    </div>
</ng-container>
```

[The next thing we learned](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"} was how to reduce the boilerplate code and multiple emission problem by using the `@InputState()` decorator that would create an
observable that was powered by the `ngOnChanges()` lifecycle hook behind the scenes.
Our previous code would evolve like this:

```typescript
// one liner to get observable input state
@InputState() private readonly inputState$!: Observable<UserPaneInputState>;

// No more setters nor BehaviorSubjects
@Input() public firstName: string;
@Input() public lastName:  string;

private readonly collapsed$$ = new BehaviorSubject<boolean>(true);

public readonly vm$ = combineLatest({
    // No distinctUntilChanged needed for input state
    inputState: this.inputState$, 
    collapsed: this.collapsed$$.pipe(distinctUntilChanged())
}).pipe(
    map(({inputState, collapsed}) => {
        return {
            firstName: inputState.firstName,
            lastName: inputState.lastName,
            collapsed
        }
    })
)

public toggleCollapse(): void {...}

```

We cleaned up the setters, BehaviorSubjects and part of the `combineLatest` boilerplate but the solution was not perfect yet.

[In the third article](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}, we learned how we could leverage ui-component state to clean this up. We created an `ObservableState` instance for every ui-component that shared the lifecycle of that component.
Meaning the `ObservableState` instance would be destroyed when its component would get destroyed. This gave us some advantages:
- Automatic cleanups, no more `takeUntil(this.destroy$$)`
- Less boilerPlate
- One **single-source-of-truth** state object to talk to
- No more BehaviorSubjects
- Reactive
- Snapshots available everywhere
- A clear distinction between ViewModels and State

Our example would look like this:

```typescript
private readonly observableState: ObservableState<UserPanelState> 
    = inject(ObservableState<UserPanelState>);
// Get observable input state
@InputState() private readonly inputState$!: Observable<UserPanelInputState>;

@Input() public firstName: string;
@Input() public lastName:  string; 

public readonly vm$ = this.observableState.state$;

public toggleCollapse(): void {
    // We don't reactivity here
    const {collapsed} = this.observableState.snapshot;
    // Patch what we need
    this.observableState.patch({collapsed: !collapsed})
}

constructor(){
    // Initialize state and connect it with
    // the reactive input state
    this.observableState.initialize({
        ...getDefaultInputState<UserPanelInputState>(this),
        collapsed: false
    }, this.inputState$)
}
```

We can see that we have dropped all the boilerplate code and we have one entity that we can interact with: The `ObservableState`.
There is a clear distinction between **InputState**, **ComponentState** and **ViewModel** which results in better separation of concerns.

## Continuing with local component state

From now on let's call **all the dynamic properties on a component the state of a component**:
- Input properties are called state from now.
- All the rest of the local properties is something we will call state from now.
- Whether we stored data in BehaviorSubject or ReplaySubject or any other kind of observable: **STATE!!**.

This part is when all the pieces of the puzzle fell together for me for the following questions:
- Why do some devs use big state management frameworks and write tons of boilerplate code for them?
- Why do some devs put everything in the store?
- Why do some devs use actions and effects?
- Why is it so hard to read this RxJS flow?
- Why is this smart component so big?
- What the hell is happening in this component?

The answer to these questions has never been more clear to me and this is the opinion of a developer(me) that has taught over 250 people (at the point of writing) how to use RxJS.
**RXJS IS HARD**. Don't shoot me, it is hard... We have:
- Transformation operators
- Combination operators
- Higher-order observable operators
- Hot observables, cold observables, warm observables
- Refcounting
- Subjects: **BehaviorSubject**, **ReplaySubject**, **Subject**, **AsyncSubject**
- Schedulers
- Connectable observables
- Take operators

Sometimes we don't know:
- When to subscribe to observables. Events could already have happened.
- When to unsubscribe to observables. We want to avoid memory leaks.
- Where to replay values, caching etc.
- Which observable triggers another observable that will eventually update some kind of local component state.
There is so much to take into account and even if you have a big expertise in RxJS, reading other developers there code can be painful when there is no real opinionated structure.

There are hundreds and hundreds of ways of creating reactive flows with RxJS and sometimes teams need **opinionated guidelines**.
I believe that's why developers use complex **@ngrx/store** flows where they put everything in one big store and try to implement the CQRS pattern in Angular by using actions and effects. I walked away from state management libraries (for most applications) 2 years after developing Angular applications full-time (the year 2018 I think), but I get why teams are using it. Even though I don't think those frameworks are the solution to most applications, they at least offer a consistent way of handling things.
I believe there are other approaches to enforce consistency and code quality in Angular applications when it comes to the local state.

**Note: Global state is something we will cover in a follow-up article**

**Should we use RxJS? I believe we do!** It's awesome, it's reactive and it's predictable... But we can easily get lost because there are too many solutions.
That's why I have created a new system that follows the principles of the previous 3 articles where we can make it easier for developers.
Before we continue, here are the requirements:
- It's small (we don't want to open-source it but maintain it ourselves) **<100 lines of code**.
- It's close to the Angular standards. No exotic solutions that diverge from the [Angular Change Detection](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"} system.
- It will make zone.js removal easy in the future.
- It's opinionated and easy to use.
- It becomes a single source of truth.
- It has snapshot functionality. We want the best of both worlds: Imperative programming and reactive programming **can go hand in hand!** 

We want to abstract that logic away from devs and make it easy for them. There are some existing solutions out there:
- [@ngrx/component-store](https://ngrx.io/guide/component-store){:target="_blank"}
- [@rx-angular/state](https://www.rx-angular.io/docs/state){:target="_blank"}

Even though there are some existing solutions out there, in the next article, we will implement a solution ourselves because:
- I don't want another Angular dependency that prevents us from updating to new Angular versions.
- I want it the integrate with the input state I wrote about earlier.
- I want to adjust as we go later on.
- I have some features in mind that I love to have specifically.
- I want something opinionated.
- It's not that much work.

## The specs of our ObservableState

[In the third article](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}, we created a class called `ObservableState`.
We can provide it in:

- Dumb components
- Smart components
- on route level (follow-up article)
- on application level (follow-up article)

We will continue to expand this specific class to achieve a fully reactive component store for every Smart and Ui component.
What this local component state should do for us:
- Avoid the need for custom `combineLatest` operators.
- Avoid the need for `distinctUntilChanged` operators.
- Avoid ```pipe(takeUntil(this.destroy))``` expressions to avoid memory leaks.
- Avoid complex RxJS flows.
- Handle replaying, ref counting for us.
- Expose a snapshot of the latest state at any time.

In the next article, we will update the `ObservableState`:
- Create `connect()` functionality: Connect observables to our `ObservableState` instance and clean up after them.
- Create `selectOnly()` functionality: Getting notified only when certain parts of the state change.
- make the state hot on initialize: Working with connectable observables to make the state hot on initialization.

We will create the implementation and some complex examples in the next article. Stay tuned!
If you can't wait, just reach out to me and I'll gladly give you my code and some examples.
Check out this article if you want to learn some [Angular State management Best practices](https://blog.simplified.courses/angular-state-management-best-practices/){:target="_blank"} 