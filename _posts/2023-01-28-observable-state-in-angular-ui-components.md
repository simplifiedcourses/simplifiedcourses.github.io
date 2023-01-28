---
layout: post
title:  "Observable state in Angular Ui components"
date:   2023-01-28
published: true
comments: true
categories: Angular, RxJS
cover: assets/observable-state-in-angular-ui-components.jpg
description: "We will dive deep into Observable state in Angular Ui components that we can use as input state and local component state"

---

What is state management? This is a very popular discussion topic in most teams.
What is state? When do we manage it? What state do we need to manage? What is managing state?
This article is about component state, which is the state that we keep inside our component.

This article is a follow-up article of the previous articles:
- [Reactive ViewModels for Ui components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}
- [Reactive input state for Angular ViewModels](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"}

## Current way of handling ViewModels

In the last article, we learned how we could create a ViewModel for Angular by calculating the input state in a one-liner:

```typescript
export class PagerComponent {
    @Input() public itemsPerPage: number = 0;
    @Input() public total: number = 0;
    @Input() public pageIndex: number = 0;

    // One-liner!
    @InputState() private readonly inputState$!: Observable<PagerInputState>;

    public readonly vm$: Observable<ViewModel> = this.inputState$.pipe(
        map(({ itemsPerPage, total, pageIndex }) => calculateVm)
    );
}
```

While this is convenient, it is not perfect just yet... There are still some issues:
- When we would set the **@Input()** properties from within this class, the reactive input state would not update when setting them.
- We have a way to handle the input state reactively but the component usually also has other states (in many situations these are handled with a number of **BehaviorSubjects** or regular object properties).
- It would be great to have one Observable state instance that lives along with this component and gets destroyed when this component gets destroyed. (Not having to implement the `ngOnDestroy()` life cycle hook ourselves.)
- There is no way to get a current snapshot of the states (other than `subject$$.value` which can feel hacky).

## Component state

The component state does not need to be compared to an application-wide or even module-wide state. 
The state doesn't need to be shared between components or even libs in our codebase.

When we think about it, every component has state:
- A property when a pane is collapsed or not
- An observable holding a list of users coming from the backend
- A FormControl containing some kind of value
- The current page index of a pager
- The params of the `ActivatedRoute` that we will use to determine the outcome of the application

Definition component state:
**Component state contains everything that determines the outcome of your page**.

Definition ViewModel:
**A ViewModel is a reactive object that contains all properties and only the properties that the template of a component needs, it is a calculation based on the component state**.

Every time an Angular (or React, Vue, Qwik, ...) application is written, the components have state.
It's just a matter of how we manage it. Some (uncontrolled and not opinionated) ways are:
- Managing plain objects in private and public properties, some read-only, some not.
- Managing **ReplaySubjects**, **BehaviorSubjects** and other kind of Observables in private and public properties.
- Extracting all the state or parts of the state into stores and making sure these states get invalidated at the right time.
- In more complex components this can result in complex RxJS logic.

In this article we suggest an opinionated and structured way that works for us:
- All the component state is kept in an observable state object.
- That observable state object holds a **BehaviorSubject** behind the scenes so we can always take snapshots when we like.
- That observable state is provided on the component and will thus get destroyed if the component gets destroyed.
- That observable state will automatically be fed by the **Input state** we were talking about earlier.
- Setting state and reading state should always be done through this reactive state object:
  - Future proof if we want to go zoneless later
  - Consistent
  - Clear
  - Separation of concerns

In this article, we will focus on component state for **ui components**. If you don't know the difference between smart and ui components you could read [this article](https://blog.simplified.courses/smart-components-ui-components-and-sandbox-facades-in-angular/){:target="_blank"}.

### Why not use an existing library for this?

There are some libraries out there that give us some of the functionality that we are looking for. After doing some research we still decided to write it ourselves. One of the best practices we teach at our training is: **Be careful which and how many dependencies you install and especially be careful of Angular dependencies**. Every dependency is a tiny marriage, a tiny contract that makes you attached to the dependency in some way. We are suggesting a different approach today: Learn why we are using this approach by reading these articles, Copy-paste the code (it is really small), and maintain it yourself. We could open-source this but we don't have time to maintain it, nor will we add features for you.
We will update these articles though.

This is why we wrote it ourselves:
- It's not that much code
- It's easy to write
- Copy-paste it and maintain it ourselves. Our code is free for you to use and it's not that complex
- Don't be blocked by new Angular versions. Dependencies are not always up to date with the latest Angular versions
- Stay close to Angular, to the standard.
- Add features ourselves when we feel we need them.
- Do we want a third-party dependency to infect every component in our codebase?

## The pager component

The pager we have written in the previous article is fully functional, but let's add some local state to introduce our first observable state.
In the following example, we have added a button that can show a dropdown of items per page.
Whether this dropdown is shown or not is considered local state and listens to the `moreOptions` property which defaults to false. (dropdown is hidden by default)
Another piece of state is the `itemsPerPageOptions` property which contains a list of numbers that the dropdown can contain.

```typescript
export class PagerComponent {
    public moreOptions = false; // Local state
    public itemsPerPageOptions = [10,20,50,100]; // Local state

    // Input state calculated by the inputs
    @InputState() private readonly inputState$!: Observable<PagerInputState>;

    @Input() public itemsPerPage: number = 0;
    @Input() public total: number = 0;
    @Input() public pageIndex: number = 0;
    @Output() public readonly pageIndexChange = new EventEmitter<number>();
    @Output() public readonly itemsPerPageChange = new EventEmitter<number>();

    public readonly vm$: Observable<ViewModel> = this.inputState$.pipe(...);

    ...

    public next(vm: ViewModel): void {
        this.pageIndexChange.emit(vm.pageIndex + 1);
    }

    public previous(vm: ViewModel): void {
        this.pageIndexChange.emit(vm.pageIndex - 1);
    }

    public goToEnd(vm: ViewModel): void {
        this.pageIndexChange.emit(Math.ceil(vm.total / vm.itemsPerPage) - 1);
    }

    // Toggle the local state
    public toggleMoreOptions(): void {
        this.moreOptions = !this.moreOptions;
    }
}

```

You can check the [Stackblitz example here](https://stackblitz.com/edit/angular-ivy-2rcext?file=src%2Fapp%2Fpager%2Fpager.component.ts)

### Some issues

Before we continue to improve this code let's pinpoint some issues:
- Setting **@Input()** properties will not update the `inputState` and thus will not update the ViewModel.
- To make the `moreOptions` part of the ViewModel we have to create a **BehaviorSubject** otherwise it won't be reactive.
- We have a partially observable state (input state) and partially plain objects (not observable).
- The template should only consume state from the ViewModel so the `moreOptions` and `itemsPerPageOptions` properties should not be public members on the class instance but be exposed through the ViewModel.
- The `toggleMoreOptions()` implementation would not work in a zone-less solution since it is not reactive.
- We want one state object that we can talk to (one single source of truth regarding state).
- The `next`, `previous` and `goToEnd` all get passed the ViewModel as a parameter so they would get access to the updated properties and we even put the `pageIndex` and `itemsPerPage` on the ViewModel for convenience reasons, even though the template doesn't need those.

## Refactor the pager component

We will dive into the implementation of the observable state later but we already know we want to provide an instance of it on the `PagerComponent` and we want to inject it as well.
That way we can implement automatic teardown logic since the observable state instance will get destroyed together with the
instance of the `PagerComponent`. Let's update the `ViewModel` and `PagerInputState` types as well and let's create a new type called `PagerState` which extends the `PagerInputState` (input state will always be part of the state).

```typescript
type ViewModel = {
    itemFrom: number;
    itemTo: number;
    total: number;
    previousDisabled: boolean;
    nextDisabled: boolean;
    moreOptions: boolean; // new property
    itemsPerPageOptions: number[]; // new property
    // pageIndex: number; (shouldn't be part of ViewModel)
    // itemsPerPage: number; (shouldn't be part of ViewModel)
};

// Covered in previous article
type PagerInputState = {
    itemsPerPage: number;
    total: number;
    pageIndex: number;
};

// Combination of PagerInputState and local component state
type PagerState = PagerInputState & {
    moreOptions: boolean;
    itemsPerPageOptions: number[];
}

@Component({
    ...
    // Provide the instance here, so it shared the life cycle of
    // PagerComponent
    providers: [ObservableState],
})
export class PagerComponent {
    // Created in previous article
    @InputState() private readonly inputState$!: Observable<PagerInputState>;

    // Inject a type safe component state
    private readonly componentState: ObservableState<PagerState> 
        = inject(ObservableState<PagerState>)
```

The next thing we want to do is make all the **Input()** properties read-only and initialize the state in the constructor.
Wince we want to only talk to the state in the future we don't want to set **Input()** properties ourselves anymore.
Since the state will be a **BehaviorSubject** behind the scenes we will need to pass it an initial state.
We have provided a `getDefaultInputState()` function that will create the default input state automatically for us.
The `initialize()` function of our component will take the initial state and an optional input state observable:

```typescript
export class PagerComponent {
    // Created in previous article
    @InputState() private readonly inputState$!: Observable<PagerInputState>;
    
    // Inject a type safe component state
    private readonly componentState: ObservableState<PagerState> 
        = inject(ObservableState<PagerState>)

    // All inputs are readonly
    // we can update them through componentState.patch({})
    @Input() public readonly itemsPerPage: number = 0;
    @Input() public readonly total: number = 0;
    @Input() public readonly pageIndex: number = 0;

    ...

    constructor(){
        // Determine the initial state
        const initialState: PagerState = {
            moreOptions: false, // Local state
            itemsPerPageOptions: [10,20,50,100], // Local state
            ...getDefaultInputState<PagerInputState>(this)
        }
        this.componentState.initialize(initialState, this.inputState$)
    }
}
```

The next thing we want to do is tackle the calculation of the ViewModel. This will not be calculated on `this.inputState$` anymore but it will get 
calculated on `this.componentState.state$`, which will hold all the state of the component. One **single source of truth of state** for this component only.
We can also see that the `itemsPerPage` and `pageIndex` aren't needed anymore on the ViewModel (they are available through the component state and I was never meant to be part of the ViewModel anyway since the template never needed access to it).
On the other hand, 2 more options are added to our ViewModel: `moreOptions` and `itemsPerPageOptions`. Since they are part of the state they are reactive
by default and could be updated through a `patch()` method.

```typescript
public readonly vm$: Observable<ViewModel> = this.componentState.state$.pipe(
    map(({ itemsPerPage, total, pageIndex, moreOptions,itemsPerPageOptions }) => {
        return {
        total,
        // itemsPerPage, (not needed anymore)
        // pageIndex, (not needed anymore)
        previousDisabled: pageIndex === 0,
        nextDisabled: pageIndex >= Math.ceil(total / itemsPerPage) - 1,
        itemFrom: pageIndex * itemsPerPage + 1,
        itemTo:
            pageIndex < Math.ceil(total / itemsPerPage) - 1
            ? pageIndex * itemsPerPage + itemsPerPage
            : total,
        moreOptions, // New
        itemsPerPageOptions // New
        };
    })
);
```

The template does not need to pass `vm` to the methods `next()`, `previous()` and `goToEnd()` anymore. The class instance now has access to the component state. The component state gives us a convenient **snapshot** that we can consume at anytime. The component is reactive but at the same time can give us a snapshot so we don't have to subscribe all the time:

```typescript
public next(): void {
    // Take what we need from the snapshot
    const {pageIndex} = this.componentState.snapshot;
    this.pageIndexChange.emit(pageIndex + 1);
}

public previous(): void {
    // Take what we need from the snapshot
    const {pageIndex} = this.componentState.snapshot;
    this.pageIndexChange.emit(pageIndex - 1);
}

public goToEnd(): void {
    // Take what we need from the snapshot
    const {total, itemsPerPage} = this.componentState.snapshot;
    this.pageIndexChange.emit(Math.ceil(total / itemsPerPage) - 1);
}
```

Since the `moreOptions` and `itemsPerPageOptions` are now living on the ViewModel we can consume them easily in a reactive way:

```html
<button ...>
    {%raw%}{{vm.moreOptions? 'Less': 'More'}}{%endraw%}
</button>
<select ...
    *ngIf="vm.moreOptions">
    <option 
        ...
        *ngFor="let option of vm.itemsPerPageOptions">...
    </option>
</select>
```

The last thing we need is to implement the `toggleMoreOptions()` function. For that we can use the `patch()` method that takes an object
that will update only the parts of the state that need to get updated:

```typescript
public toggleMoreOptions(): void {
    const {moreOptions} = this.componentState.snapshot;
    this.componentState.patch({moreOptions: !moreOptions});
}
```

## What did we solve?

Before continuing on to the actual implementation of the observable state, let's summarize what we have fixed.
- We have a reactive state model **with a snapshot** that is updated every time an input changes.
- The reactive state model can be patched manually.
- We have a clear ViewModel that does not contain anything but the data our template needs.
- The template does not need to pass the ViewModel back to the class instance.
- We have a single source of truth when it comes to handling state.
- We can consume the input state observable easily in the `initialize()` method.
- Our local state is also reactive and consumable through the ViewModel
- We separated ViewModel logic from state logic completely
- We don't have to worry about implementing the `ngOnDestroy()` life cycle hook.

## Implementing the Observable state

In this specific article, we will have a look at a limited version of the observable state. An implementation that contains only 55 lines of code (without comments).
Let's summarize what kind of functionality this observable state already holds for us:
- It should clean itself up, when the instance that provides the `ObservableState` gets destroyed it should destroy the `ObservableState` as well.
- It should hold a **BehaviorSubject** behind the scenes that is exposed as an observable that only emits when the state is initialized.
- It should `distinctUntilChanged()` on all the properties and thus only emit when one of the values has actually changed.
- It should expose a **snapshot** of the current state.
- The `intialize()` method should take the default state but should also take an input state observable as a second argument and feed the state accordingly in a safe way (no memory leaks).
- It should be completely type-safe.
- It should be possible to patch values as a partial object so that there is only one emission on the state object for every time we call `patch()` and not for every key.

### What we will not implement today
The implementation of this article is limited to the set of features listed above and will not contain:
- `connect({users: users$})`: a method to connect multiple observables to our observable state.
- `onlySelect(['foo', 'bar'])`: a method that will return an observable that only is updated with and when one of the passed keys of our state gets updated.

```typescript
@Injectable()
export class ObservableState<T extends Record<string, any>>
    implements OnDestroy
{
    private readonly destroy$$ = new Subject<void>();
    // The core of our state
    private readonly state$$ = new BehaviorSubject<T | null>(null);
    public readonly state$ = this.state$$.pipe(
        filter((v) => v !== null),
        map((v) => v as T),
        distinctUntilChanged((previous: T, current: T) => {
            const keys = Object.keys(current);
            return keys.every((key) => {
            return current[key] === previous[key];
            });
        }),
        takeUntil(this.destroy$$) // clean up
    );

    public get snapshot(): T {
        // Make ts compiler happy and tell developer
        // when they forgot to initialize
        if (!this.state$$.value) {
            throw new Error('State not initialized yet');
        }
        return this.state$$.value;
    }

    public initialize(state: T, inputState$?: Observable<Partial<T>>): void {
        this.state$$.next(state);
        if (inputState$) {
            inputState$.pipe(takeUntil(this.destroy$$))
            .subscribe((res: Partial<T>) => this.patch(res));
        }
    }

    public patch(object: Partial<T>): void {
        // Make ts compiler happy and tell developer
        // when they forgot to initialize
        if (!this.state$$.value) {
            throw new Error('State not initialized yet');
        }
        // Take all the old values from the state
        // update with values of patched state
        let newState: T = { ...this.state$$.value };
        Object.keys(object).forEach((k) => {
            newState = { ...newState, [k]: object[k] };
        });
        // only next once
        this.state$$.next(newState);
    }

    public ngOnDestroy(): void {
        // next subject when state should be destroyed
        // which avoids manual takeUntil(destroy) statements
        this.destroy$$.next();
    }
}

```

Here is an [updated Stackblitz example](https://stackblitz.com/edit/angular-ivy-b8megj?file=src%2Fapp%2Fpager%2Fpager.component.ts) that contains everything you need for these kinds of components.
## Conclusion

In this article, the observable state implementation is not yet complete, but we will update it in the next article where we will use it for more complex state management in smart components. Stay tuned