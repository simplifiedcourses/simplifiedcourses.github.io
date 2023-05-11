---
layout: post
title:  "Evolving from the SIP principle towards Observable state"
date:   2023-03-14
published: true
comments: true
categories: [Angular, RxJS, State management]
cover: assets/evolving-from-the-sip-principle-towards-observable-state/evolving-from-the-sip-principle-towards-observable-state-in-angular.jpg
description: "This article explains how I moved from the SIP principle towards Observable state and greatly simplified my code."
---

## Intro

On our legacy blog (brecht.io) we wrote an article about using [the SIP principle in Angular](https://blog.brecht.io/the-sip-principle/){:target="_blank"}.
The SIP principle is a reactive pattern that helps us to think reactively in technologies like Angular and RxJS.
It results in us splitting up all the Observables into 3 different groups of Observables:
- **S**: **Source Observables**: The Observables that will emit based on user interaction. In other words, the events in our application we want to listen to.
- **I**: **Intermediate Observables**: These Observables will get calculated by listening to the source Observables and will be used to create our presentation observables.
- **P**: **Presentation Observables**: The Observables that are needed for our template. These will get calculated based on the source Observables and presentation Observables.

## The SIP solution

In the [following StackBlitz example](https://stackblitz.com/edit/angular-drgnwm?file=src%2Fapp%2Fapp.component.ts){:target="_blank"}, we refactored the old application from the previous blog article towards an Angular 15 application with a recent version of RxJS.

![SIP principle starship finder](/assets/evolving-from-the-sip-principle-towards-observable-state/starship-finder.png)

The application has the following functionalities:
- Search starships by the term
- Search starships by model
- Fetch a starship by a random model
- Perform client-side filtering by the number of passengers
- Show a loading flag when something is loading
- Cancel existing calls

The **source Observables** looked like this:

```typescript
// Source Observables
public readonly selectedModel$ = new ReplaySubject<string>(1);
public readonly searchTerm$ = new ReplaySubject<string>(1);
public readonly randomModel$ = new ReplaySubject<string>(1);
public readonly numberOfPassengers$ = new BehaviorSubject<number>(1000000);
```

The **intermediate Observables** looked like this:

```typescript
// Intermediate Observables
const query$ = merge(
    this.searchTerm$, 
    this.randomModel$, 
    this.selectedModel$
).pipe(startWith(''));

const results$ = query$
    .pipe(
        switchMap(query => this.fetchData(query)),
        shareReplay(1)
    );
```

The **presentation Observables** looked like this:

```typescript
// Presentation Observables
this.loading$ = merge(
    query$.pipe(map(() => true)),
    results$.pipe(map(() => false))
);

this.filteredResults$ = combineLatest(
    results$, 
    this.numberOfPassengers$,
).pipe(
    map(([results, numberOfPassengers]) => 
        this.filterByPassengers(results, numberOfPassengers)
    )
)
```

While this might be all honkey dory for a senior RxJS developer and it's completely reactive, it might let other developers frown:
- We have to know different types of Subjects: `ReplaySubject` and `BehaviorSubject`.
- We have to know the operators: `merge`, `startWith`, `shareReplay`,  `combineLatest`.
- `combineLatest` can have multiple emissions at the same time
- The `shareReplay` could even introduce memory leaks because we forgot to pass `refCount: true`.
- It's hard to see the initial state of this component.
- It's hard to see the actual dataflow of this component.
- There are multiple ways of handling this with RxJS so it could seem strange to other developers.
- It starts to become complex.

Smart components tend to get complex quite fast and can contain tons and tons of RxJS code.
There is one statement we would like to focus on:
**RxJS isn't necessarily hard, but code written with RxJS can become very hard very fast.**.

We believe that's the reason why libraries like [rx-angular/state](https://www.rx-angular.io/docs/state){:target="_blank"} and [@ngrx/component-store](https://ngrx.io/guide/component-store){:target="_blank"} are created. While Angular is evolving rapidly and we tend to avoid third party dependencies for state management, we wrote a simple and small solution ourselves.
Check out this article if you want to learn some [Angular State management Best practices](https://blog.simplified.courses/angular-state-management-best-practices/){:target="_blank"} 

## The Observable state solution

The previous code has some downsides:
- You have to know RxJS extensively.
- "6-months ago you", might get lost and so might your colleagues.
- It's not very opinionated.
- We are not managing component state very well, it's shattered across the file, across ReplaySubjects, a BehaviorSubject and other Observables.

In this article, we will refactor the previous code to use [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"}.
This article is a follow-up article of the previous articles (newest to oldest):
- [Observable component state in Angular](https://blog.simplified.courses/observable-state-in-smart-components/){:target="_blank"}
- [Observable state for ui components in Angular](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}
- [Reactive input state for Angular ViewModels](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"}
- [Reactive ViewModels for Ui components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}

In the first article, we started out by creating ViewModels, which can replace the presentation Observables. In other words: ViewModels would replace the **P** in **SIP**.
After that, we created component state for ui components and now we will simplify this rather complex RxJS code by using the [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"}.

The [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} is a custom-written implementation that we started to write in [this article](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}.

Some characteristics of [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"}:
- It's compact!
- It's simple!
- It requires an initial state (default state).
- It's a **BehaviorSubject** on steroids behind the scenes.
- It cleans up subscriptions automatically.
- It enforces queue scheduling.
- It makes the state **hot** the moment we call the `initialize()` method.
- It makes sure you don't need any state management dependencies.
- We can connect all kinds of Observables.
- It exposes a snapshot and a `patch()` method that updates partial state.
- It exposes a `state$` observable that we can subscribe to.

Let's start by creating a `AppComponentState` type and extending the `AppComponent` from [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"}.
```typescript
type AppComponentState = {
    // source state
    query: string;
    numberOfPassengers: number;
    // intermediate state
    starships: any[];
    // presentation state
    loading: boolean;
    filteredStarships: any[];
}
...
export class AppComponent  extends ObservableState<AppComponentState> {
}
```

This will give the component access to the following methods/properties:

- **initialize(defaultState):** This initializes the state with default values
- **patch(partial):** This gives us the ability to patch one or multiple pieces of the state in one go.
- **onlySelectWhen(keys):** This will return the state as an observable only when one of the properties bound to the passed keys changes.
- **connect(keyObservableMap):** This will connect an object with observables to the actual state. It will subscribe only once, next to local BehaviorSubject and unsubscribe when needed.
- **state$:** This returns the state as an observable, that will get unsubscribed automatically on `ngOnDestroy`.
- **snapshot**: This returns the current snapshot of our state.

Let's initialize the state by calling the `initialize()` method from [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} to initialize the state with default values:

```typescript
export class AppComponent  extends ObservableState<AppComponentState> {
    ...
    constructor() {
        super()
        this.initialize({
            query: '',
            numberOfPassengers: 100000,
            starships: [],
            loading: true,
            filteredStarships: []
        })
    }
}

```


This seems pretty straightforward: We have a clean type for the entire state of this component and we have initialized the [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} with
default values. Next, let's create 2 methods to update the query and number of passengers. As you can see we can update multiple pieces of state in one go, avoiding too many emissions.
In the `changeQuery()` method, we not only want to set the `query` but also the `loading` property:

```typescript
public changeQuery(query: string): void {
    // patch 2 properties in one patch
    this.patch({query, loading: true});
}

public changeNumberOfPassengers(numberOfPassengers: number): void {
    this.patch({numberOfPassengers})
}
```

Let's dive a bit deeper into the asynchronous stuff.
We know the starships should be loaded when the `query` changes, so we want to get notified when the `query` property of the state changes.
For that, we can use the `onlySelectWhen()` method that is provided by [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"}.
This method will return the entire state when one of the properties bound to the passed keys changes.
When the `fetchData()` call is finished we can leverage the `tap()` operator to create a side effect that uses the `patch()` method to put the loading property of our state object to `false`.

```typescript
constructor(){
    super();
    this.initialize({...});

    // fetch starships ONLY when query changes
    const starships$ = this.onlySelectWhen(['query']).pipe(
        switchMap(({query}) => this.fetchData(query)),
        tap(() => {
            // set loading back to false
            this.patch({loading: false})
        })
    )
    // connect it to the state
    this.connect({
      starships: starships$
    })
}
```

The `connect()` method will subscribe only once and communicate with the private BehaviorSubject of [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} so we don't have to worry about multicasting anymore. This means we don't ever have to write `shareReplay({refCount: true, bufferSize:1})` again.
Did we mention we don't ever have to write `takeUntil(this.destroy$$)` anymore either?

We have calculated and connected the `starships` state, but we still need to calculate the `filteredStarships` state.
This piece of state depends on the `starships` state and `numberOfPassengers` state (which is set by the `changeNumberOfPassengers()` method).
We can easily calculate that observable, by using the `onlySelectWhen()` method again...
In the `connect()` method we can see that we can just pass the `filteredStarships$` Observable and bind it to the `filteredStarships` property:

```typescript
 constructor() {
    super()
    this.initialize({});
    const starships$ = this.onlySelectWhen(['query']).pipe(...)
    // calculate the filteredStarships ONLY when starshiops and numberOfPassengers change
    const filteredStarships$ = this.onlySelectWhen(['starships', 'numberOfPassengers']).pipe(
      map(({starships, numberOfPassengers}) => this.filterByPassengers(starships, numberOfPassengers))
    )
    this.connect({
      starships: starships$,
      // Just connect it to the state
      filteredStarships: filteredStarships$
    })
  }
```

This is the only reactive logic we need, and we don't even need the `combineLatest` operator anymore!
We can just return the state if one of the **presentation states** change and create an encapsulated ViewModel for our component:

```typescript
public readonly vm$ = this.onlySelectWhen([
    'loading',
    'filteredStarships',
    'numberOfPassengers',
  ]).pipe(
    map(({ loading, filteredStarships, numberOfPassengers }) => ({
      loading,
      filteredStarships,
      numberOfPassengers,
    }))
  );

```

By doing that our template gets cleaned up quite nicely as well:

```html
<ng-container *ngIf="vm$|async as vm">
    <sidebar class="sidebar" 
      [models]="fixedModels" 
      [numberOfPassengers]="vm.numberOfPassengers"
      (search)="changeQuery($event)"
      (selectModel)="changeQuery($event)"
      (randomModel)="changeQuery($event)"
      (changeNumberOfPassengers)="changeNumberOfPassengers($event)"
    >
    </sidebar>
    <div class="main">
      <starship-list 
          [starships]="vm.filteredStarships"
          [loading]="vm.loading">
      </starship-list>
    </div>
  </ng-container>
```

You can find the entire working solution in [this StackBlitz example](https://stackblitz.com/edit/angular-rgfbr9?file=src%2Fapp%2Fapp.component.ts){:target="_blank"}

### Why didn't we open-source this?

Well, we kinda did... It's in the stackblitz, right? We just don't maintain it for you. We advise our clients to **not install every npm package that they find**, and we allow you to own this small piece of code yourself.
It's not complex, it's easy to understand, easy to maintain, and you can add functionality as much as you want.
We will see in a followup article, how we can leverage this principle to create very simple **global state**, where we can optimize the lifecycle by using the **dependency injection system**
that Angular offers us. There is no need to go to complex state management libraries that request a lot of boilerplate and maintenance.

### But what about Signals?

Angular Signals will greatly improve the readability of reactive programming and will make **state management** with RxJS obsolete in a way.
This doesn't mean that we don't need RxJS anymore. It means that state management might become way easier with Signals.
Signals work synchronously and this is exactly why our [ObservableState](https://github.com/simplifiedcourses/observable-state/) exposes a snapshot.
Since Signals are coming in the future of Angular and refactoring from this approach towards Signals is a breeze we consider it a valid approach for current Angular development.

To make you all completely happy, we have taken the liberty to refactor this approach towards signals:
```typescript
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
```

Since Signals are a thing of the future, and we just want to showcase what we know now, we will not dive deeper in this solution and we do not
recommend to use Signals in production code until they have reachted a stable state.

Checkout the [StackBlitz example with Signals](https://stackblitz.com/edit/angular-rjwjeq?file=src%2Fapp%2Fapp.component.ts){:target="_blank"}

## Wrapping up

We learned that even the SIP principle doesn't fix all the reactive complexity for us.
We have implemented our own solution that we maintain ourselves when it comes to simple state management.
By using [ObservableState](https://github.com/simplifiedcourses/observable-state/) we have:
- introduced queue scheduling
- made the knowledge on hot/vs cold observables obsolete
- made sure we never have to do `takeUntil(this.destroy$$)`
- an opinionated way of managing state
- a clear overview which state lives in our component
- avoided a bunch of operators and RxJS logic

Signals are awesome, but they are not ready yet!
Thanks for reading and stay awesome!