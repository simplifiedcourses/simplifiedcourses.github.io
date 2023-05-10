---
layout: post
title:  "Observable state in Angular"
date:   2023-05-10
published: true
comments: true
categories: [Angular, State management]
cover: assets/observable-state-in-angular.jpg
description: "What is Observable State and how do we use it?"
---

This article is a follow-up article of these previous articles (newest to oldest):
- [Evolving from the SIP principle towards Observable state](https://blog.simplified.courses/evolving-from-the-sip-principle-towards-observable-state/){:target="_blank"}
- [Observable component state in Angular](https://blog.simplified.courses/observable-state-in-smart-components/){:target="_blank"}
- [Observable state for ui components in Angular](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}
- [Reactive input state for Angular ViewModels](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"}
- [Reactive ViewModels for Ui components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}

**I also gave a talk at ng-be where I introduced everything in 25 minutes.** You might want to check out [this video](https://www.youtube.com/watch?v=58h_w7PzNtM){:target="_blank"} before continuing.

In this article, we will tackle [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"}.
We will discuss the API and why we wrote something like that.
We will also cover some use cases on when to use it.

We already covered [ui-component state](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"} where we manage the local state of a ui-component in an opinionated way.
We combined that state and connected it with [input state](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"} (which is the state that is passed by `@Input()` properties).
After that, we covered [Smart component state](https://blog.simplified.courses/evolving-from-the-sip-principle-towards-observable-state/){:target="_blank"} where we avoid RxJS complexity and make state management more junior-friendly.

## Explaining the Observable state API

ObservableState is a class that is under 100 lines of code, and the goal is simple.
**Abstracting RxJS logic**

While it still allows you to write as much RxJS logic as you want, `ObservableState` handles the following for you:
- Multicasting
- Cleaning up subscriptions
- Distinct changes
- Snapshot functionality
- Initializing the state in one method
- Patching multiple pieces of state, without triggering state more than once
- Calculating pieces of state based on other pieces of state
- Picking pieces of global state
- Getting notified when pieces of state get updated
- Selecting one piece of state
- Connecting observables and feeding your state automatically

**All of this functionality is achieved in under 100 lines of code** and it does all that in a BehaviorSubject behind the scenes. 

### Creating/initializing observable state

To create a State component we can simply do the following:

```typescript
type MyComponentState = {
    foo: string;
    bar: string;
}
...
class MyComponent extends ObservableState<MyComponentState> {
    constructor(){
        super();
        this.initialize({
            foo: '',
            bar: ''
        })
    }
}
```

We have extended from `ObservableState` and passed a type called `MyComponentState`, to make the whole thing type-safe in the constructor, we have initialized our state with the `initialize()` method that lives on `ObservableState`.
Our state is now fully initialized.

When we would want to create a shared State that can be provided anywhere in Angular we could develop it like this:

```typescript
type ApplicationState = {
    foo: string;
    bar: string;
}
@Injectable()
class ApplicationObservableState extends ObservableState<ApplicationState> {
    constructor(){
        super();
        this.initialize({
            foo: '',
            bar: ''
        })
    }
}
```

If we want this state to be a singleton we could use `providedIn: 'root'` to provide the state at the local level

```typescript
...
@Injectable({providedIn: 'root'})
class ApplicationObservableState extends ObservableState<ApplicationState> {
    ...
}
```

It's a good idea to [keep the state as low as possible](https://blog.simplified.courses/angular-state-management-best-practices/#best-practice-number-3-keep-your-state-as-low-as-possible){:target="_blank"} so maybe we want to provide this at the entry component of a feature:

```typescript
@Component({
    providers: [MyFeatureObservableState]
})
class MyFeatureComponent {
    ...
}
```

Now we can consume the instance of `MyFeatureObservableState` in any child component of `MyFeatureComponent` by injecting it:

```typescript
class SomeChildComponent {
    private readonly featureState = inject(MyFeatureObservableState);
}
```

#### API

```typescript
/**
 * Observable state doesn't work without initializing it first. Our state always needs
 * an initial state. You can pass the @InputState() as an optional parameter.
 * Passing that @InputState() will automatically feed the state with the correct values
 * @param state
 * @param inputState$
 */
public initialize(state: T, inputState$?: Observable<Partial<T>>): void;
```

Connecting an `inputState$` is explained in depth [in this article](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}:

### Patching pieces of state

When using multiple BehaviorSubjects, you might end up patching more of them in parallel.

**Check out this bad example for instance:**

```typescript
// Bad example
class UserComponent {
    firstName$$ = new BehaviorSubject('Brecht');
    lastName$$ = new BehaviorSubject('Billiet');
    fullName$ = combineLatest({firstName: this.firstName$$, lastName: this.lastName$$}]).pipe(
        map(({firstName, lastName}) => firstName + ' ' + lastName)
    );
    setName(firstName: string, lastName: string): void {
        this.firstName$$.next(firstName);
        this.lastName$$.next(lastName);
    }
}

```
When we call `setName()` in the previous example the `fullName$` Observable will be triggered twice at the same time.

When using `ObservableState` we don't have that issue since we have a patch function that takes a partial of the state.
We could solve the previous problem like this.
**This is a better solution:**
```typescript
// Good example
class UserComponent extends ObservableState<UserComponentState> {
    constructor(){
        super();
        this.initialize({
            firstName: 'Brecht',
            lastName: 'Billiet'
        })
    }
    setName(firstName: string, lastName: string): void {
        // Patch multiple properties in one go
        this.patch({ firstName, lastName })
    }
}
```

#### API

```typescript
 /**
 * Patch a partial of the state. It will loop over all the properties of the passed
 * object and only next the state once.
 * @param object
 */
public patch(object: Partial<T>): void;
```

### The state$ Observable

The whole thing is called `ObservableState` so that means that every class that extends from it has access to a `state$` property.
We can use that `state$` property to calculate a [ViewModel](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"} or simply feed our template:

```typescript
// Good example
class UserComponent extends ObservableState<UserComponentState> {
    // Calculate ViewModel based on state$ Observable
    vm$ = this.state$.pipe(state => {
        return {
            name: state.firstName + ' ' + state.lastName
        }
    });
    constructor() { ... }
    setName(firstName: string, lastName: string): void { ... }
}
```

#### API

```typescript
/**
 * Return the entire state as an observable
 * Only use this if you want to be notified on every update. For better optimization
 * use the onlySelectWhen() method
 * where we can pass keys on when to notify.
 */
public readonly state$: Observable<T>;
```

### Snapshot

Every class that extends `ObservableState` now also has access to a `snapshot` property that holds the latest value of the state.
Let's say that we want to implement a `capitalizeName()` function where we would need access to the current state:

```typescript
class UserComponent extends ObservableState<UserComponentState> {
    constructor(){...}
    setName(firstName: string, lastName: string): void {...}
    
    capitalize(): void {
        // Pick what we need from the snapshot
        const { firstName, lastName } = this.snapshot;
        // Patch the state in one go
        this.patch({ firstName: firstName.toUpperCase(), lastName: lastName.toUpperCase() });
    }
}
```

This is way more convenient than subscribing to a piece of state in combination with `take(1)` or `withLatestFrom`.

#### API

```typescript
/**
* Get a snapshot of the current state. This method is needed when we want to fetch the
* state in functions. We don't have to use withLatestFrom if we want to keep it simple.
*/
public get snapshot(): T;
```

### Connecting state

Now we can initialize state, patch state and get a snapshot and that's great... But we also want to work with asynchronous state.
Every class that extends `ObservableState` now has access to a `connect()` method where we can pass an object with observables it should subscribe to:

```typescript
constructor(){
    super();
    this.connect({
        user: this.userService.fetchUser(this.activatedRoute.snapshot.userId),
        time: interval(1000).pipe(map(() => new Date().getTime()),
        searchQuery: this.searchControl.valueChanges,
    });
}
```

This `connect()` method handles a lot for us:
- Unsubscription logic
- Multicasting, no more manual `shareReplay({bufferSize: 1, refCount: true})
- `distinctUntilChanged` on every property
- It will patch the state automatically

**Don't forget!** that state has a `state$` Observable we can subscribe to, but also has a convenient snapshot that we can use.
So this piece of code is hideous:

```typescript
// bad
addAddress(address: Address): void {
    this.user$
        .pipe(
            take(1),
            takeUntil(this.destroy$$),
            switchMap(user => {
                this.userService.addAddress(user.id, address);
            })
        )
        .subscribe()
}
```
becomes a lot cleaner:
```typescript
// good
addAddress(address: Address): void {
    this.userService.addAddress(user.id, this.snapshot.address).subscribe();
}
```

Also, you can connect it to `@ngrx/store` observables as well if you have those in your workspace:
```typescript
constructor(){
    super();
    this.connect({
        user: this.store.select(selectUser())
    });
}
```

#### API

```typescript
/**
 * This method is used to connect multiple observables to a partial of the state
 * pass in an object with keys that belong to the state with their observable
 * @param object
 */
public connect(object: Partial<{ [P in keyof T]: Observable<T[P]> }>): void;
```

### Picking state

When we would advise you to let most components extend from `ObservableState` because we want to treat our components as tiny state machines, we would probably end up with a bunch of
global observable states as well. That's why we have provided a `pick()` method where you can pick whatever you want from whichever state.

Let's say that we have a global `ShoppingCartState`:

```typescript
type ShoppingCartState = {
    entries: ShoppingCartEntry[];
    persisted: boolean;
    products: []
}
class ShoppingCartObservableState extends ObservableState<ShoppingCartState> {
    ...
}
```

In our `ProductDetailComponent` we need access to the `entries` and `products` of the `ShoppingCartState`. 
First of all, we would need the initial values of these properties so we can pass them in the `initialize()` method.
We can use the `snapshot` property from `ObservableState` for that:

```typescript
class ProductDetailComponent extends ObservableState<ProductDetailComponentState> {
    // Inject the external state
    shoppingCartState = inject(ShoppingCartObservableState)
    constructor(){
        super();
        // Pick the pieces from the snapshot
        const { entries, products } = this.shoppingCartState.snapshot;
        this.initialize({
            product: null,
            loading: false,
            entries,
            products
        })
    }
}
```
Now we have taken the initial values, but the `entries` and `products` are not connected yet.
To connect this, we can use the `pick()` method:

```typescript
class ProductDetailComponent extends ObservableState<ProductDetailComponentState> {
    ...
    constructor(){
        super();
        // Pick the pieces from the snapshot
        const { entries, products } = this.shoppingCartState.snapshot;
        this.initialize({...})

        this.connect({
            // Pick observables from another state and connect them
            ...this.shoppingCartState.pick(['entries', 'products'])
        })
    }
}
```

The `pick()` method will return an object of observables, so by using the spread operator we can reduce this into a one-liner.

#### API

```typescript
/**
 * Pick pieces of the state and create an object that has Observables for every key that is passed
 * @param keys
 */
public pick<P>(keys: (keyof T)[]): Partial<{ [P in keyof T]: Observable<T[P]> }>;
```

### Selecting a piece of state

Every class that extends from `ObservableState` also has a `select()` method where we can pass a key that is part of our state.
This is convenient if we want to get an Observable from one piece of state:

```typescript
firstName$ = this.select('firstName');
```

#### API
```typescript
/**
 * Returns an observable of a specifically selected piece of state by a key
 * @param key
 */
public select<P extends keyof T>(key: P): Observable<T[P]>;
```

### Calculating state based on another state

This is one of the most powerful features of `ObservableState`. It provides us with an `onlySelectWhen()` method that returns the entire state
when specific properties in that state get a new value.
When one of the properties in the state connected to those passed keys will get a new value, the Observable will emit.
We can use that to **calculate some state based on other state**.

Let's take this more complex example. We have a `ProductOverviewSmartComponent` that shows some products but has client-side paging and client-side filtering.
First, we have to load the products, and then based on the `products` and `query` we would have the calculate the `filteredProducts`. When the `filteredProducts`, the `pageIndex` or the `itemsPerPage` changes, we would have to calculate the `pagedProducts`. This requires us to calculate state based on state that was already calculated by other state.
Seems complex right?! Well, with  `ObservableState` it's easy as pie.
In the following example ,we can see that initialize the state with initial default state, and we connect `products` with the `getProducts()` method of a `productService`.

```typescript
type ProductOverviewState = {
    pageIndex: number;
    query: string;
    itemsPerPage: number;
    products: Product[];
    filteredProducts: Product[];
    pagedProducts: Product[];
}

@Component({...})
export class ProductOverviewSmartComponent extends ObservableState<ProductOverviewState> {
    constructor() {
        super();
        this.initialize({
            pageIndex: 0,
            itemsPerPage: 5,
            query: '',
            products: [],
            filteredProducts: [],
            pagedProducts: [],
        });

        this.connect({
            products: this.productService.getProducts()
        })
    }

}
```

The `connect()` method will subscribe to the Observable that `getProducts()` returns, and will multicast it for us.
Let's start by calculating the  `filteredProducts` :

```typescript
type ProductOverviewState = {...}

@Component({...})
export class ProductOverviewSmartComponent extends ObservableState<ProductOverviewState> {
    constructor() {
        super();
        this.initialize({ ... });
        // When products or query changes, recalculate filteredProducts
        const filteredProducts$ = this.onlySelectWhen(['products', 'query']).pipe(
            map(({ products, query }) => {
                return products.filter(p => p.name.toLowerCase().indexOf(query.toLowerCase()) > -1)
            })
        );
        ...
    }
}
```

In the next sample we will see that it will calculate `pagedProducts` every time the `filteredProducts`, the `pageIndex` or the `itemsPerPage` state changes:
```typescript
type ProductOverviewState = {...}

@Component({...})
export class ProductOverviewSmartComponent extends ObservableState<ProductOverviewState> {
    constructor() {
        super();
        this.initialize({ ... });
        // When products or query changes, recalculate filteredProducts
        const filteredProducts$ = this.onlySelectWhen(['products', 'query']).pipe(
            map(({ products, query }) => {
                return products.filter(p => p.name.toLowerCase().indexOf(query.toLowerCase()) > -1)
            })
        );
        // When filteredProducts, pageIndex or itemsPerPage changes, recalculate filteredProducts
        const pagedProducts$ = this.onlySelectWhen(['filteredProducts', 'pageIndex', 'itemsPerPage']).pipe(
            map(({ filteredProducts, pageIndex, itemsPerPage }) => {
                const offsetStart = (pageIndex) * itemsPerPage;
                const offsetEnd = (pageIndex + 1) * itemsPerPage;
                return filteredProducts.slice(offsetStart, offsetEnd);
            })
        );
        ...
    }
}
```

The only thing left to do is connect the `filteredProducts$` and `pagedProducts$` Observables to the state in the `connect()` method:

```typescript
type ProductOverviewState = {...}

@Component({...})
export class ProductOverviewSmartComponent extends ObservableState<ProductOverviewState> {
    constructor() {
        super();
        this.initialize({ ... });
        // When products or query changes, recalculate filteredProducts
        const filteredProducts$ = this.onlySelectWhen(['products', 'query']).pipe(...);
        // When filteredProducts, pageIndex or itemsPerPage changes, recalculate filteredProducts
        const pagedProducts$ = this.onlySelectWhen(['filteredProducts', 'pageIndex', 'itemsPerPage']).pipe(...);

        // Connect what is calculate, and feed the state automatically
        this.connect({
            products: this.facadeService.getProducts(),
            filteredProducts: this.filteredProducts$,
            pagedProducts: this.pagedProducts$,
        })
    }
}
```

#### API

```typescript
/**
 * Returns the entire state when one of the properties matching the passed keys changes
 * @param keys
 */
public onlySelectWhen(keys: (keyof T)[]): Observable<T>;
```

That's it. You can find a complete example [here](https://github.com/simplifiedcourses/observable-state/blob/main/src/demo-app/components/smart/product-overview/product-overview.smart-component.ts){:target="_blank"}. If you clone the repo, you can run it with:

```shell
npm run api
npx nx run shop:serve
```

## Wrapping up

Sometimes it's better to own something yourself than to install third-party libraries. `ObservableState` can be easily copy-pasted into your project and it is only 100 lines of code to maintain. Own it, improve it, the source code is really not that hard.
At the time of writing this, I have a role as **Frontend Architect** at **DHL Express** (the aviation branch) and most of our codebases use `ObservableState` which evolved into happy developers and predictable code.
We have ditched 90% of the RxJS complexity, improved performance and have a very junior-friendly entry-level in our workspaces.

Some examples:
- [Pager with Observable state](https://stackblitz.com/edit/angular-ivy-b8megj?file=src%2Fapp%2Fpager%2Fpager.component.ts){:target="_blank"}
- [Starship finder with Smart Observable State](https://stackblitz.com/edit/angular-rgfbr9?file=src%2Fapp%2Fapp.component.ts){:target="_blank"}

If you need more examples, please reach out to me. I'm happy to help.

Ps: We are also working on a `SignalState` which follows the same principles as `ObservableState`, but specifically for Angular Signals: You can check the first version of an example of the use [here](https://github.com/simplifiedcourses/observable-state/blob/main/src/demo-app/components/smart/product-overview-signal/product-overview-signal-smart.component.ts){:target="_blank"}

Special thanks to the awesome reviewers:
- [Tim Deschryver](https://timdeschryver.dev/){:target="_blank"}
- [Mihai Paraschivescu](https://twitter.com/mikeandtherest){:target="_blank"}
- [Daniel Glejzner](https://twitter.com/DanielGlejzner){:target="_blank"}
- [Fabian Gosebrink](https://twitter.com/FabianGosebrink){:target="_blank"}