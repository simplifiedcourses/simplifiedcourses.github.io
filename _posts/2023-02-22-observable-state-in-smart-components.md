---
layout: post
title:  "Observable state in Angular Smart components"
date:   2023-02-22
published: false
comments: true
categories: Angular, RxJS
cover: assets/observable-state-in-angular-ui-components.jpg
description: "We will dive deep into Observable state in Angular Smart components that we can use to create better reactive flows and local state management"

---

This article is a follow-up article of the previous articles (newest to oldest):
- [Observable state for ui component in Angular](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}
- [Reactive input state for Angular ViewModels](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"}
- [Reactive ViewModels for Ui components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}

## Quick recap

We started with creating reactive [ViewModels for UI components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}. We learned how to keep our template clean and have a fully observable model that we
can feed to the template (a specific reactive model for the template of a specific component). ViewModels are derived from local properties on that component and its `@Input()` properties. If we wanted them to be reactive we had to create BehaviorSubjects for all of those properties that we later combined into a ViewModel by using the `combineLatest` operator.
In the following example, we see all the boilerplate we need to create a ViewModel for the properties: `firstName`, `lastName` and `collapsed`.

```typescript
private readonly firstName$$ = new BehaviorSubject<string>('');
@Input() public set firstName(v: string) {
    this.firstName$$.next(v);
}

private readonly lastName$$ = new BehaviorSubject<string>('');
@Input() public set lastName(v: string) {
    this.lastName$$.next(v);
}
private readonly collapsed$$ = new BehaviorSubject<boolean>(true);

public toggleCollapse(): void {
    this.collapsed$$.next(!this.collapsed$$.value); // dirty
}

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

The good thing is the template looks way cleaner: We can extract logic from the template into the ViewModel and we only have one subscription.

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

@Input() public firstName: string; // no more setters
@Input() public lastName:  string; // no more setters

private readonly collapsed$$ = new BehaviorSubject<boolean>(true);

public readonly vm$ = combineLatest({
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

[In the third article](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}, we learned how we could leverage ui-component state to clean this up. We created an `ObservableState` instance for every ui component that shared the lifecycle of that component.
Meaning the `ObservableState` instance would be destroyed when its component would get destroyed. This gave us some advantages:
- Automatic cleanups, no more `takeUntil(this.destroy$$)`
- Less boilerPlate
- One **single-source-of-truth** state object to talk to
- No more BehaviorSubjects
- Snapshots available everywhere
- A clear distinction between ViewModels and State

Our example would like like this:

```typescript
private readonly observableState: ObservableState<UserPanelState> = inject(ObservableState<UserPanelState>);
// one liner to get observable input state
@InputState() private readonly inputState$!: Observable<UserPanelInputState>;

@Input() public firstName: string; // no more setters
@Input() public lastName:  string; // no more setters

public readonly vm$ = this.observableState.state$;

public toggleCollapse(): void {
    const {collapsed} = this.observableState.snapshot;
    this.observableState.patch({collapsed: !collapsed})
}

constructor(){
    this.observableState.initialize({
        ...getDefaultInputState<UserPanelInputState>(this),
        collapsed: false
    }, this.inputState$)
}
```

## Continuing with local component state

We can see that we have dropped all the boilerplate and we have one entity that we can talk to: The `ObservableState`.
From now on let's call all the dynamic properties on a component state:
- Input properties are state
- All the rest of the local properties are state
- Whether BehaviorSubject or ReplaySubject or any other kind of observable: **state**

We already went a way