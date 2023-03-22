---
layout: post
title:  "Global Observable state in Angular"
date:   2023-03-21
published: true
comments: true
categories: Angular
cover: assets/global-observable-state-in-angular.jpg
description: "How do we handle global Observable state in Angular"
---

This article is a follow-up article of the previous articles (newest to oldest):
- [Evolving from the SIP principle towards Observable state](https://blog.simplified.courses/evolving-from-the-sip-principle-towards-observable-state/){:target="_blank"}
- [Observable component state in Angular](https://blog.simplified.courses/observable-state-in-smart-components/){:target="_blank"}
- [Observable state for ui components in Angular](https://blog.simplified.courses/observable-state-in-angular-ui-components/){:target="_blank"}
- [Reactive input state for Angular ViewModels](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"}
- [Reactive ViewModels for Ui components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}

In this article, we will tackle **reusable global Observable state** in Angular.

## Why not use a state management framework?

Why aren't we advocating for frameworks like **@ngrx/store** or **ngxs**? Because at Simplified Courses we are big believers in the **KISS** principle. We like to avoid:
- Boilerplate
- Complexity 
- Big dependencies if we can avoid them
- Putting every piece of data in the store 
- Managing state that we shouldn't manage
- State diverging from what is in the backend

We love to go for the pragmatic approach and not use a store as a communication tool.

We have to ask ourselves if it makes sense to store **EVERYTHING** in a single store?
We have to ask ourselves why we can't call an AJAX service and get the response back directly in our component when using @ngrx/store all the way. 

**@ngrx/store approach**: 
- Create action type, action, effect, reducer, selector
- Send action
- Consume effect
- Perform XHR call
- If success: Put result in store
- If failure: Put error in store
- Execute reducers
- Execute selector to fetch the result or the error
- On destruction of the component: Send an action, that removes the data from the store.

**KISS approach**:
- Call a service
- Consume the data
- (nothing to invalidate)

The second step seems easier, and it is, but it sometimes requires us to write a bunch of RxJS code. We believe that state management frameworks gain popularity because RxJS code tends to become hard over time, and teams need a consistent way of handling communication, but this article isn't about advising you not to use state management frameworks. I wrote this article because I want to show a way to make state management simple without the need for a big dependency.

## Reasoning about state in general

It doesn't matter if we use a state management framework, BehaviorSubjects or something else. The reasoning of state management should stay the same.
This boils down to a few best practices:

### Rule 1: Keep state as low as possible

Can you keep it in your lowest component in the component tree? **Go for it!** Do you need to share it between 2 components? **Keep the state in their parent component**.
Do you need to share state between different routes but in the same lib? Go ahead and **keep the instance of that state in that lib**. Do you have state at the application level? Keep the instance of that state as a singleton. The higher we go in the hierarchy, the less state we should have.

The lower you keep the state:
- The fewer elements (components, services, directives) in your application the state will impact
- The more standalone the instantiator of our state will be
- The easier it will be to manage that state
- The easier it will be to clean up that state: When you keep your state on your component, your state will get destroyed the moment your component gets destroyed

### Rule 2: Don't store state that you don't need to store

This mostly refers to data state. We could consider data as state, but that doesn't mean we need to store it. In some cases we do, and in some cases, we don't. Storing too much data state could result in:
- Store data diverging from database data
- Extra logic/complexity/maintenance of code
- The inability to get responses from AJAX calls because we put everything in a store

## Global state

In previous articles, we have covered local component state and smart component state. This article is about global state.

In [this repository](https://github.com/simplifiedcourses/observable-state){:target="_blank"} we are exposing some functionality that everyone can use:
- `ObservableState`
- `@InputState()`

Angular Signals will probably impact state management a lot, but since we can not use it in production yet, we still need some kind of state management thing if we don't want to rely on BehaviorSubjects. That's why we have written `ObservableState` which has these interesting methods:

### initialize

```typescript
initialize(initialState: T, inputState$?: Observable<InputState>): void
```

This method needs the initial state as the first parameter, state always needs an initial value remember? As the second and optional parameter it takes `inputState$` which is explained [here in depth](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/#optimizing-with-a-typescript-decorator){:target="_blank"}.

We can not use ObservableState methods before this `initialize()` method is called.

### patch

```typescript
patch(partialState: Partial<T>): void
```

This takes a part of the state and is used to update multiple properties of the state at the same time.

### onlySelectWhen

```typescript
onlySelectWhen(keys: (keyof T)[]): Observable<T>
```

This handy method will return an Observable of the entire state but only emits when one of the passed keys updates in the state. This method is ideal to calculate pieces of state based on other state.

```typescript
const filteredUsers$ = this.onlySelectWhen(['query', 'users'])
    .pipe(map(({query, users}) => {...}))
```

### connect

```typescript
connect(obj: {[key: keyof T]: Observable}): void
```

This connects an object with observables to the state. We can pass any observable here, and it will automatically subscribe to the observables and unsubscribe when the state gets destroyed. I
We can use it to store calculated pieces of state:

```typescript
this.initialize({
    users: [],
    query: '',
    filteredUsers: []
})
const filteredUsers$ = this.onlySelectWhen(['query', 'users'])
    .pipe(map(({query, users}) => {...}))
this.connect({filteredUsers: filteredUsers$})
```

n the following example we can just pass it a timer and a coordinates observable:

```typescript
this.connect({
    time: interval(1000).pipe(map(() => new Date().getTime())),
    coords: fromEvent(document, 'mousemove').pipe(
        map((v: any) => ({ x: v.offsetX, y: v.offsetY }))
    ),
});

```

### pick

```typescript
pick(keys): {[key: keyof T]: Observable}
```

This method is important when picking pieces of state from somewhere else.
```typescript
this.connect({
    ...this.applicationObservableState.pick(['sidebarCollapsed','currentUser']);
})
```


## Shopping cart state

Here is an example of a **shopping cart** state:

```typescript
type ShoppingCartState = {
    entries: ShoppingCartEntry[]
}

@Injectable({
    // we can provide it in root, or on route level
    // or on smart component level, or on
    // ui component level
    providedIn: 'root'
})
// exending from ObservableState will make ShoppingCartObservableState injectable
// everywhere and will add the default ObservableState logic to this class
export class ShoppingCartObservableState extends ObservableState<ShoppingCartState> {
  constructor() {
    super();
    // Initializing with initial state is a best practice in all
    // state management frameworks
    this.initialize({
      entries: []
    })
  }

  public addToCart(entry: ShoppingCartEntry): void {
    const entries = [...this.snapshot.entries, entry];
    this.patch({ entries });
  }

  public deleteFromCart(id: number): void {
    const entries = this.snapshot.entries
        .filter(entry => entry.productId !== id);
    this.patch({ entries });
  }

  public updateAmount(id: number, amount: number): void {
    const entries = this.snapshot.entries
        .map(item => item.productId === id ? { ...item, amount } : item);
    this.patch({ entries });
  }
}
```

We can see that we have added 3 convenient methods:
- `addToCart()`
- `deleteFromCart()`
- `updateAmount()`

These methods just use the `patch()` method and the `snapshot` getter that `ObservableState` provides, which gives us a convenient way of managing state:

By using Angular's dependency injection, this way of creating state classes becomes very powerful since this state instance would get destroyed when the component or module that is providing this class is destroyed.
This gives us great control over the lifecycle of this piece of state.

## Picking from other pieces of state

Let's say that we have a `ProductsObservableState` that holds products and categories:

```typescript
type ProductsState = {
    products: Product[];
    categories: Category[];
}
@Injectable({
    providedIn: 'root'
})
export class ProductsObservableState extends ObservableState<ProductsState> {
    private readonly productsService =  inject(ProductsService);
    constructor() {
        super();
        this.initialize({
            products: [],
            categories: []
        });
        this.connect({
            products: this.productsService.getAll(),
            categories: this.productsService.getAllCategories()
        })
    }
}
```

We can now see that we can use the `connect()` method to easily fetch and manage the state
of products and categories.

When we would want to use the products and categories in our ShoppingCartService we could use the `pick()` method to achieve that:

```typescript
type ShoppingCartState = pick(ShoppingCartState, 'products'|'categories') & {
    entries: ShoppingCartEntry[]
}
...
export class ShoppingCartObservableState extends ObservableState<ShoppingCartState> {
    private readonly productsObservableState = inject(ProductsObservableState);
  constructor() {
    super();
    const { products, categories } = this.productsObservableState.snapshot;
    this.initialize({
      entries: [],
      products,
      categories
    })
    this.connect({
        ...this.productsObservableState.pick(['products','categories']);
    })
  }

  public addToCart(entry: ShoppingCartEntry): void {...}

  public deleteFromCart(id: number): void {...}

  public updateAmount(id: number, amount: number): void {...}
}
```

