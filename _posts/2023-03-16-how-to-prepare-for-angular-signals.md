---
layout: post
title:  "How to prepare for angular signals"
date:   2023-03-16
published: true
comments: true
categories: [Angular, State management]
cover: assets/how-to-prepare-for-angular-signals.jpg
description: "Angular signals are coming. How can we prepare?"
---

Lately, Twitter is blowing up when it comes to Angular Signals.
A Signal is a **reactive primitive** that will be used to simplify reactive programming in Angular.
Currently, most of the applications running in production heavily rely on RxJS or state management frameworks to achieve reactivity.
Some developers even store all of the local component state and server responses in a store, to ensure a consistent way of reactive communication.
This way the store becomes a communication tool and blocks direct access to the responses of our XHR calls from within our components.
This results in boilerplate, complexity and we prefer the KISS approach. At Simplified Courses we tend to avoid State management frameworks. Check out this article if you want to learn some [Angular State management Best practices](https://blog.simplified.courses/angular-state-management-best-practices/){:target="_blank"} 
We believe that Signals will make reactivity a lot easier.

Signals will be in **developer preview** in **Angular 16** but we have no idea when it will be completely stable.
We can already start playing with Signals in Angular if we install **Angular version@16.0.0-next.1** but we can
not use Angular Signals yet in our production code because:
- The api is not set in stone yet.
- Angular hasn't optimized [Change Detection](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"} yet.
- There will be a lot of different code changes needed to integrate Signals with the framework completely: Forms, Routing, etc...

It is kind of sad that we know that this awesome reactive primitive will **shape the future of Angular** in terms of simple reactivity and optimized change detection, but we can't use it just yet.
So, let's see how we can prepare for this awesome feature already today.

## How can we prepare for Signals today?

### Play with it!

First of all, I would recommend everyone to play with Signals so we can get used to them and see the simplicity it brings to the table. This can greatly impact our mindset in terms of reactive programming.
In the following example, I refactored an application that was using the [SIP principle](https://blog.simplified.courses/evolving-from-the-sip-principle-towards-observable-state/){:target="_blank"} towards [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} and later to [Signals](https://stackblitz.com/edit/angular-rjwjeq?file=src%2Fapp%2Fapp.component.ts){:target="_blank"}.

The Signals version of the code looks like this:

```typescript
export class AppComponent {
    ...

    private readonly loading = signal(true);
    private readonly numberOfPassengers = signal(100000);
    private readonly query = signal('');
    // We still need RxJS for asynchronous stuff
    private readonly results = fromObservable(
        fromSignal(this.query).pipe(
        switchMap((query) => this.fetchData(query)),
        tap(() => this.loading.set(false))
        ),
        []
    );

    private readonly filteredStarships = computed(() =>
        this.filterByPassengers(this.results(), this.numberOfPassengers())
    );
    public readonly viewModel = computed(() => {
        return {
            filteredStarships: this.filteredStarships(),
            numberOfPassengers: this.numberOfPassengers(),
            loading: this.loading(),
        };
    });

    public changeQuery(query: string): void {
        this.loading.set(true);
        this.query.set(query);
    }

    public changeNumberOfPassengers(numberOfPassengers: number): void {
        this.numberOfPassengers.set(numberOfPassengers);
    }
    ...
  }
}

```
We are not going to dive deep into this code, since we already did that in the [previous article](https://blog.simplified.courses/evolving-from-the-sip-principle-towards-observable-state/){:target="_blank"}.

But as we can see there is not much RxJS complexity going on here. There are no:
- `combineLatest()` operators.
- Multicasting operators like `share()`, `shareReplay(1)`, `shareReplay({refCount: true, bufferSize: 1})` or even `connectable()`.
- No different types of subjects.
- `takeUntil(this.destroy$$)` operators to implement to avoid memory leaks.
- `withLatestFrom()` operators to retrieve the current value of a piece of state.

What should become clear here is that:
- Every Signal, meaning every piece of state has **an initial value**. That's a very healthy way of thinking about state in general.
- We still need RxJS to implement `switchMap()` functionality. Asynchronous things like `debounceTime()` will also still be useful.
- We use `fromObservable()` and `fromSignal()` to combine the best of both worlds: We think about **state as something synchronous** and we think
about the real **RxjS logic as something asynchronous**.
- We have a clean **ViewModel** that exposes a specific Signal that is only meant to be used by the template.
- Setting and retrieving states are done synchronously instead of asynchronously. (by executing the signal or calling `set`)

### Think about reactive component state and pass initial values

Avoid component properties like `isOpen: boolean` **or any other plain javascript objects or values**.
Instead, think about reactive component state. We want to store all the state in reactive objects because that's the only way
we can go zoneless in the future. Since we don't have Signals yet we advise you to use **BehaviorSubjects**.
BehaviorSubjects are reactive, they have an initial value and they always replay the last value.
This means that BehaviorSubjects are reactive, but also always have a snapshot since it provides us with a `value` property.

```typescript
public isOpen = false; // Not reactive
public isOpen$$ = new BehaviorSubject<boolean>(false);// Reactive
```

BehaviorSubjects also enforce initial values, just as signals expect initial values. This again is one step closer to the future.
We would recommend to abstract the BehaviorSubjects away behind a [custom state implementation](https://stackblitz.com/edit/angular-rgfbr9?file=src%2Fobservable-state.ts){:target="_blank"} that handles simple state management for us. In the following example we use that custom state implementation.
As we can see here, we have a clear overview of our component state and we have initialized all the state of our component with initial data:

```typescript
// We have forced ourselves to think about the local state
// of the application by creating a type for it
type AppComponentState = {
  query: string;
  numberOfPassengers: number;
  starships: any[];
  loading: boolean;
  filteredStarships: any[];
};

export class AppComponent extends ObservableState<AppComponentState> {
    constructor() {
        super();
        // We have forced ourselves to think about the initial state
        this.initialize({
            query: '',
            numberOfPassengers: 100000,
            starships: [],
            loading: true,
            filteredStarships: [],
        });
    }
}
```

### Avoid withLatestFrom and combineLatest

The `withLatestFrom` operator tends to create complexity in reactive flows, while we only want to have the latest value of a piece of state.
With Signals we won't need this operator anymore as we could just execute the Signal to get the latest data.
By teaching our mindset to always have access to the latest value of a piece of state we make the switch towards Signals easier.

Also, **BehaviorSubjects** offer a `value` property or `getValue()` method.
The following piece of code starts to become complex, because the difference between `combineLatest()` and `withLatestFrom()` matters a lot.
We want to recalculate `res$` every time `foo$$` **or** `bar$$` changes, but we need the latest value of `baz$$`.
We are trying so hard to be reactive here that we forget we could just get the value from `baz$$`:

```typescript
private readonly foo$$ = new BehaviorSubject(0);
private readonly bar$$ = new BehaviorSubject(1);
private readonly baz$$ = new BehaviorSubject(2);
public readonly res$ = combineLatest([this.foo$$, this.bar$$]).pipe(
    withLatestFrom(this.baz$$),
    switchMap(([[foo, bar], baz]) => {
        return this.fetch(foo, bar, baz)
    })
)

// expose the BehaviorSubject states to the template
public readonly foo$ = this.foo$$.asObservable();
public readonly bar$ = this.bar$$.asObservable();
public readonly baz$ = this.baz$$.asObservable();
```

This snippet (without the `withLatestFrom()` operator, for instance would be less complex:

```typescript
const foo$$ = new BehaviorSubject(0);
const bar$$ = new BehaviorSubject(1);
const baz$$ = new BehaviorSubject(2);
const res$ = combineLatest([foo$$, bar$$]).pipe(
    switchMap(({[foo, bar]}) => {
        return this.fetch(foo, bar, this.baz$$.value); // get the value directly
    })
)
...
```

If we would use [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} which is explained in depth in [this article](https://blog.simplified.courses/evolving-from-the-sip-principle-towards-observable-state/){:target="_blank"}, the code would look even cleaner.

```typescript
constructor(){
    super();
    this.initialize({
        foo: 0,
        bar: 1,
        baz: 2
    });
    const res$ = this.onlySelectWhen(['foo', 'bar').pipe(
        switchMap(({foo, bar}) => {
            return this.fetch(foo, bar, this.snapshot.baz)
        })
    )
    this.connect({res: res$}); // store the result in our local component state
}
// state$ is being exposed to the template by default
```
We wouldn't need to worry about multicasting or unbsubscribing. The scheduling would be automatically set to `queueScheduler` and it would distinct values for us.
There is no more `combineLatest()` (which could have multiple emissions at the same time) and we clearly see that if `foo` **or** `bar` changes, we have to fetch
some data. The `snapshot` getter of [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} would give us the value just as a signal would do.
The previous piece of code would look like this when using Signals:

```typescript
public readonly foo = signal(0);
public readonly bar = signal(1);
public readonly baz = signal(2);
public readonly res$ = fromSignal(computed(() => ({foo: this.foo(), bar: this.bar()}))).pipe(
    switchMap(({foo, bar}) => {
        return this.fetch(foo, bar, this.baz())
    })
);
public readonly res = fromObservable(this.res$);

// all the signals are available to the template here
```

The code itself isn't 100% the same. But the principles and mindset kind of are...

### ViewModels, avoid async pipes

Another step towards signals would be to remove as many async pipes as possible, while still keeping the component reactive.
For that reason, we could **limit the amount of async pipes of a component to one** and use a ViewModel.
With the BehaviorSubject approach, we could do this:

```typescript
private readonly foo$$ = new BehaviorSubject(0);
private readonly bar$$ = new BehaviorSubject(1);
private readonly baz$$ = new BehaviorSubject(2);
private readonly res$ = combineLatest([foo$$, bar$$]).pipe(
    switchMap(({[foo, bar]}) => {
        return this.fetch(foo, bar, this.baz$$.value)
    })
)
public readonly vm$ = combineLatest({
    foo: this.foo$$.pipe(distinctUntilChanged()), 
    bar: this.bar$$.pipe(distinctUntilChanged()),
    baz: this.baz$$.pipe(distinctUntilChanged()),
    res: this.res$.pipe(distinctUntilChanged())
})
```

The ViewModel of course only needs to contain the state that our template needs.

The template would look like this:

```html
<ng-container *ngIf="vm$|async as vm">
    {%raw%}{{vmm.foo}}
    {{vmm.bar}}
    {{vmm.baz}}
    {{vmm.res}}{%endraw%}
</ng-container>
```

As we can see, there is only one async pipe and refactoring the template to use signals would be as easy as changing the  `vm$|async` to `vm()`:

```html
<ng-container *ngIf="vm() as vm">
    {%raw%}{{vmm.foo}}
    {{vmm.bar}}
    {{vmm.baz}}
    {{vmm.res}}{%endraw%}
</ng-container>
```


To make this complete we will showcase how this example would look like with [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} which is basically one **BehaviorSubject** on steroids behind the scenes:

```typescript
constructor(){
    super();
    this.initialize({
        foo: 0,
        bar: 1,
        baz: 2
    });
    const res$ = this.onlySelectWhen(['foo', 'bar').pipe(
        switchMap(({foo, bar}) => {
            return this.fetch(foo, bar, this.snapshot.baz)
        })
    )
    this.connect({res: res$}); // store the result in our local component state
}
public readonly vm$ = this.state$;
```

In the future, using Signals, the code would look like this:

```typescript
private readonly foo = signal(0);
private readonly bar = signal(1);
private readonly baz = signal(2);
private readonly res$ = fromSignal(computed(() => ({foo: this.foo(), bar: this.bar()}))).pipe(
    switchMap(({foo, bar}) => {
        return this.fetch(foo, bar, this.baz())
    })
);
private readonly res = fromObservable(this.res$);
public readonly vm = computed(() => {
    return {
        foo: this.foo(),
        bar: this.bar(),
        baz: this.baz(),
        res: this.res()
    }
})
```

It's worth mentioning that libraries like [rx-angular/state](https://www.rx-angular.io/docs/state){:target="_blank"} and [@ngrx/component-store](https://ngrx.io/guide/component-store){:target="_blank"}
follow similar patterns and also enforce you to keep your component state inside a reactive store.
These libraries would certainly bring us closer to Signals but we try to keep it simple and avoid third-party state management libraries unless we really need them.

## Wrapping up

Signals are awesome, and while it looks attractive to use them, it might be a little bit too soon for that.
That doesn't mean we can't prepare for them! By following these principles we can refactor towards Signals in the future without too much effort:
- Always keep your component state in a reactive object.
- Always initialize your component state with initial state, **Signals also need initial state**.
- Try to make the distinction between state and asynchronous flows. **Signals => State => synchronous**, **Rxjs => asynchronous flows**.
- Embrace the fact that you have access to the latest value at all times. Use a BehaviorSubject or something else that provides us with snapshots.
- Avoid using too many **async** pipes by embracing the ViewModel principle

Hope you enjoyed it. Comment if you liked it!


