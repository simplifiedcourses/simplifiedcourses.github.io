---
layout: post
title:  "Introducing State management with Angular SignalState"
date:   2023-07-12
published: false
comments: true
cover: assets/reorder-conditional-table-columns-in-angular.jpg
categories: [Angular, State management, ObservableState, Angular Signals]
description: "This article teaches you how to manage state in Angular applications in an opinionated way with Signals"
---

## Some history

We have been focused on making Angular State Management easier and developer friendly for years now.
Some of the problems we were trying to fix were:
- Maintaining consistency
- Avoiding boilerplate code
- Avoiding complexity
- Avoiding dependency hell between all different kinds of states
- Avoiding learning all the ins and outs of RxJS (Subjects, Schedulers, Multicasting, Error handling, ...)
- Dealing with colleagues their RxJS code or code that we have written ourselves 6 months ago

A while ago we wrote an article about [Angular State Management Best Practices](https://blog.simplified.courses/angular-state-management-best-practices/){:target="_blank"} that can be wrapped up in:
- Keep it simple, stupid.
- Don't manage state you shouldn't manage.
- Keep your state as low as possible.
- Keep state synchronous.
- Always provide snapshots.
- Always provide the initial state.
- Use immutable data structures.

The first thing that we did was **stop using state management frameworks** to avoid boilerplate and complexity. We created Angular Injectable Services instead that used **BehaviorSubjects** behind the scenes because:
- They are reactive.
- They have an initial value.
- We can combine pieces of state with `combineLatest()` and create standalone state instances that were framework agnostic.
- We could provide these instances on any component level, but also as a singleton with `provideIn: 'root'`.
- It was simply a lot less code to write and maintain.

We didn't have to write actions, action types, reducers, selectors, effects and more importantly, we didn't have to maintain them.

This being said **the BehaviorSubject service approach** had some downsides:
- They were brittle and unopinionated
- We had to know RxJS extremely well
- We had to worry about multicasting
- We had to worry about unsubscribing from observables
- When using `combineLatest()` the observable could be triggered multiple times per change detection cycle
- Snapshots were only available in BehaviorSubjects, not in the calculated state
- We couldn't patch multiple BehaviorSubjects at the same time

It resulted in instances that contained BehaviorSubjects and RxJS complexity some of the time.

## ObservableState

To fix all these problems we created [ObservableState](https://blog.simplified.courses/categories/observablestate/){:target="_blank"}. ObservableState is a <100 lines of code class that does not contain any complexity but solves a lot of issues for us:
- It abstracts RxJS complexity away
- It unsubscribes for us
- It handles multicasting for us
- It exposes our state as an Observable
- It exposes a snapshot of our state
- It's extremely typesafe
- We can just connect observables with a simple method
- It can be used for component state and global state
- It's super small and easy to understand
- we can still combine it with RxJS as much as we want
- It's junior friendly
- We can provide it on every level of the dependency injection tree
- It's easy to create [ViewModels](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"} from them

Behind the scenes, ObservableState is a **BehaviorSubject on steroids** that makes state management in Angular much easier. You can copy-paste it in your project, the code is easy to understand/maintain and you can adapt the code as you like, without having to rely on third-party state management frameworks that could make your application more complex.

The core principles of ObservableState are:
- You always need to initialize your state with default values
- The state should be synchronous
- The state should be reactive but also provide a snapshot
- We should be able to calculate state by other calculated state
- Every ObservableState instance that uses the state from another ObservableState instance should listen to it and copy that state inside the local state

To find a complete API explanation of ObservableState, check out [this article](https://blog.simplified.courses/observable-state-in-angular/){:target="_blank"}

## Signals

There is no way around it, **Signals are coming** and some benefits are:
- Better Change Detection
- KISS
- Synchronous
- They need initial data
- We can read the value at all times (snapshot)
- Easy to create ViewModels from

Creating a ViewModel from signal states is easy as pie:

```typescript
vm = computed(() => {
    return {
        foo: this.foo(),
        bar: this.bar(),
        baz: this.baz(),
        res: this.res()
    }
})
```

Read this article if you want to [prepare for Angular Signals](https://blog.simplified.courses/how-to-prepare-for-angular-signals/){:target="_blank"}

Using ObservableState will actually prepare us for Signals. We are using it at our customer **DHL Express aviation** in different projects and it not only boosted productivity and DX, but it also followed the main principles of Signals:
- KISS
- Synchronous
- Initial data
- Snapshots
- Easy to extract ViewModels from

Refactoring a ViewModel from an observable to a signal is as easy as this:

```html
<!-- Observables !-->
<ng-container *ngIf="vm$|async as vm">
    {{vmm.foo}}
    {{vmm.bar}}
    {{vmm.baz}}
    {{vmm.res}}
</ng-container>
<!-- Signals !-->
<ng-container *ngIf="vm() as vm">
    {{vmm.foo}}
    {{vmm.bar}}
    {{vmm.baz}}
    {{vmm.res}}
</ng-container>
```

As we can see we just have to replace the `$|async` with `()`.

Signals could be seen as simplified BehaviorSubjects that are baked into the framework but that doesn't mean we don't need opinionated state management anymore.

## Angular SignalState

While we can go a long way with using Signals in our components and creating Injectable services with signals it's not a complete solution for State Management in Angular.

Components with functionality always contain some state, whether it is a few simple properties or some reactive objects like an Observables or Signals, they always contain some state that will change over time. For that reason, we are firm believers that **Every component should be treated as a state machine**! Whether you use ObservableState or @ngrx/component-store.

Since the patterns of ObservableState really shined in our recent **DHL projects** and it turns our components into state machines, we decided to create a **SignalState**.
SignalState doesn't use a single BehaviorSubject behind the scenes but it used multiple signals. Every piece of state has its signal behind the scenes. This is a decision we did not take lightly and we choose to have multiple signals because we have a lot of calculated state that can be calculated based on another piece of state that is calculated. Having one signal introduced too many issues.

### How do we use it?

#### Creating the state type

Since we aim for type-safety and want a neat overview of what our state hold we still create our state types like this (We even pick from a global state here):

```typescript
type StarshipFinderComponentState = {
  numberOfPassengers: number;
  starships: any[];
  counter: number;
  loading: boolean;
  filteredStarships: any[];
} 
// pick from a global state
& Pick<ModelsState, 'models' | 'query'>;
```

#### Turning our components into a state machine

To use the SignalState on a component, instead of doing:

```typescript
export class StarshipFinderComponent extends ObservableState<StarshipFinderComponentState> {
    ...
}
```

We can now do:
```typescript
export class StarshipFinderComponent extends SignalState<StarshipFinderComponentState> {
    ...
}
```

#### Initializing the state

Just like with ObservableState, initializing the state can be done in one place like this:

```typescript
export class StarshipFinderComponent extends SignalState<StarshipFinderComponentState> {
    // Inject some global SignalState
    private readonly modelsSignalState = inject(ModelsSignalState);
    constructor() {
        super();
        // Take some default state from a globalState called modelsSignalState
        const { query, models } = this.modelsSignalState.snapshot;
        // Initialize the state
        this.initialize({
            query,
            models,
            numberOfPassengers: 100000,
            starships: [],
            loading: true,
            filteredStarships: [],
            counter: 0,
        });
    }
 }
```

#### Creating a ViewModel

Creating a ViewModel becomes a breeze too:

```typescript
public readonly vm = this.selectMany([
    'loading',
    'filteredStarships',
    'numberOfPassengers',
    'models',
    'counter',
]);
```
Because of the fact that SignalState is very type-safe, the `vm` object will only expose the keys that are passed in the array of the `selectmany()` method. This results in better DX.

If our ViewModel is more complex we can pass the `selectMany()` method a mapping function:
```typescript
public readonly vm = this.selectMany([
    'loading',
    'filteredStarships',
    'numberOfPassengers',
    'models',
    'counter',
], ({loading, filteredStarships, numberOfPassengers, models, counter}) => {
    // add some calculation logic here
    return {...}
});
```

#### RxJS interoperability and calculating state

RxJS interoperability, calculating derived pieces of state becomes easy as well:
```typescript
const starships = toSignal(
    toObservable(this.select('query')).pipe(
        // sometimes we still need RxJS
        debounceTime(200),
        switchMap((query) => this.fetchData(query))
    ),
    { initialValue: [] }
);

const filteredStarships = this.selectMany(
    ['starships', 'numberOfPassengers'],
    ({ starships, numberOfPassengers }) =>
        this.filterByPassengers(starships, numberOfPassengers)
    );
```

The `starships` signal is based on the `query` signal here and does some RxJS logic because we need to debounce and perform an XHR call.
the `filtererdStarships` signal is dependent on the freshly calculated `starships` signal but it also depends on the `numberOfPassengers` signal.

#### Connecting signals to the store

This is one of the biggest advantages of not having a huge amount of uncontrolled signals lying around. We want to connect an object with Signals so our store gets fed automatically.

Instead of using the `connect()` method of ObservableState to connect observables, we now can use the `connect()` method to connect signals, like the ones we just calculated now:

```typescript
  this.connect({
      starships,
      filteredStarships,
    });
```

We can even pick pieces of state from another SignalState so our local state machine is getting fed automatically:

```typescript
this.connect({
    // listen to the state of modelsSignalState 
    // and push the values of query and models into the localstate
    ...this.modelsSignalState.pick(['query', 'models']),
    starships,
    filteredStarships,
});
```

#### Patching state and using snapshots

Just like with ObservableState we can use the `patch()` method to patch multiple properties:

```typescript
public changeQuery(query: string): void {
    this.patch({ loading: true, query: string });
}
```
This is cleaner than manually patching a bunch of different signals here.

Getting an overview of the entire state can be easily done by using `snapshot`:

```typescript
if(this.snapshot.query = ''){
    this.patch({query: 'foo})
}
```

There is also a reactive `state` signal:

```typescript
effect(() => {
    console.log(this.state())
})
```

## The implementation

We are still implementing as we speak but you can find the current implementation in [this StackBlitz example](https://stackblitz.com/edit/stackblitz-starters-8wuadn?file=src%2Fsignal-state.ts){:target="_blank"}:

We will cover the implementation in a next article when it is stable and more battle-tested. Remember, ObservableState is battle-tested and refactoring to SignalState will be a breeze.

## The app

The app can be found [here](https://stackblitz.com/edit/stackblitz-starters-8wuadn?file=src%2Fcomponents%2Fsmart%2Fstarship-finder%2Fstarship-finder.component.ts){:target="_blank"} so you can already play with it.
