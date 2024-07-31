---
layout: post
title: "Modern Angular State management with signals"
date: 2024-07-30
authors: [brecht_billiet]
published: true
cover: assets/modern-angular-state-management-with-signals.jpg
comments: false
categories: [Angular, State management, Signals]
description: "State management has evolved a lot in Angular, here is some history and some best-practices on how to manage state today."
---

State management is a topic that I love, and it is one of the most discussed topics in companies. There are various
ways of managing state, this article will go over the history of state management, and how I handled state in the past and now.
This article is about modern state management practices that can be applied without any third-party lib.

## History

State management in Angular has evolved a lot over the years. In the following steps I will explain how state management
evolved for me and which best practices I use today.

### Models

We started out with putting state in models. Models are typescript classes that contained properties of state.
The properties in those models where values and objects that were mutated.

The lack of immutability and unidirectional dataflow resulted in unexpected behavior and made it hard
to track bugs and develop in a consistent way.

### The redux pattern

After models, we started using the redux pattern, mostly with @ngrx/store, which led to a more consistent way of
handling state and forced an immutable unidirectional dataflow. To enforce consistency, we started putting all state
and data in the store.

The downside of this is the following:
- Crazy amount of work/code.
- Actions don't return a value so we **have** to put everything in the store if we want to read it.
- A lot of boilerplate code.
- Effects calling effects can be overly complex.
- Maintaining and writing tests became time intensive.

### BehaviorSubject services

As the pain of the redux pattern became to big for me personally, I evolved towards simple services that contained BehaviorSubjects.
The nice thing is that we could leverage the Angular dependency injection system to determine the lifecycle of a piece of state:
- Providing the service on component level, would share the lifecycle of that component, and would destroy the state together with that component.
- We could have state per component, per lib, and as a global singleton.

Using BehaviorSubject services made it possible to ditch state management frameworks and keep it simple.
A shopping cart state machine would look like this:

```typescript
@Injectable()
export class ShoppingCartStateMachine {
    // A BehaviorSubject behind the scenes
    readonly #entries$$ = new BehaviorSubject<ShoppingCartEntry[]>([]);
    
    // Exposed as an observable
    public readonly entries$ = this.#entries$$.asObservable();
    
    // A snapshot for easy access
    public get snapshot(): { items: ShoppingCartEntry[] } {...}
    
    // Some manipulation methods to update the state
    public addToCart(entry: ShoppingCartEntry) {...}
    public deleteFromCart(id: number): void {...}
    public updateAmount(id: number, amount: number): void {...}
}
```

This was simple, clean, but also had problems:
- It wasn't that opinionated.
- It needed a lot of know-how of RxJS.
- To calculate state based on state we needed complex `combineLatest` operators.
- Updating multiple pieces of state at the same time resulted in multiple emissions in the `combineLatest` operators.
- We needed to subscribe/unsubscribe on state.

### Frameworks evolved and I created ObservableState

Frameworks got simpler and we started removing things from the store and keep them on component level.
We had @ngrx/component-store and rx-angular playing a big role in that. They made state management on components possible.

However, I wanted to keep my dependencies to a minimum, so I created ObservableState. A lightweight 100 line class that solved the following issues for us:
- It automatically unsubscribed from observables
- It provided a snapshot
- We could update multiple pieces of state at the same time without triggering the state multiple times
- We didn't need `combineLatest` anymore
- It distinct the changes automatically
- We didn't have issues with multicasting anymore

I used ObservableState in big applications in production and it made my life easier.
I also started using the ViewModel pattern where I created a reactive observable specifically for the template.
The viewModel of a pager component could look like this:

```typescript
protected readonly vm$ = this.state$.pipe(
    map(({itemsPerPage, total, pageIndex}) => {
        return {
            ...
            nextDisabled: pageIndex >= Math.ceil(total / itemsPerPage) - 1,
            itemFrom: pageIndex * itemsPerPage + 1,
            itemTo: pageIndex < Math.ceil(total / itemsPerPage) - 1
            ? pageIndex * itemsPerPage + itemsPerPage
            : total,
        }
    }
))
```

The advantages of viewModels:
- Only one subscription (async pipe) needed.
- No more template logic, everything is in the class, and the template became extremely readable and clean.
- No more multicasting issues (multiple subscriptions triggering multiple rest calls).


### Signals

Moments before I gave my talk at NG-BE 2023 about [Reactive patterns in Angular enterprise solutions](https://blog.simplified.courses/reactive-patterns-in-angular-enterprise-solutions/){:target="_blank"}, the Angular team dropped a bomb.
**Signals**, a new reactive primitive to simplify state management. Did my talk just became obsolete?

The cool thing is, signals solved the exact same things I was solving with ObservableState:
- **No more RxJS complexity**
- No more subscribing/unsubscribing
- Easy snapshots, no more `withLatestFrom` stuff...
- Memoization
- ...

I loved signals from the start, they were so simple and elegant. I decided to create `ngx-signal-state` which again
was used in big projects for big clients (even though the npm package never really took off). `ngx-signal-state` had the same api
as ObservableState, resulting in easy refactors towards signals.

## State management in 2024

When doing software development, we don't have the luxury of putting or heads in the sand. We constantly have to zoom out
and look at the problems differently. We have to re-evaluate all the time. Frameworks evolve, so we have to adapt as well.

I sell an [Angular Boilerplate](https://www.simplified.courses/angular-boilerplate){:target="_blank"} that focuses on scalable architecture, 
supabase-integration and showcases a real app called influencer. The whole goal of this boilerplate is to let you start building
immediately. I have done all the hard work, so you can just start building your app and be productive from the beginning.
Anyway, this Boilerplate had one issue. It had a dependency on `ngx-signal-state`, because... Well, I loved managing state
with that library. However, people that want to buy my boilerplate might not want to learn `ngx-signal-state`.
Having a dependency like that in a boilerplate that you sell, is not an advantage.


### Dropping ngx-signal-state

Just for fun, I started refactoring pages towards signals instead of using signal state.
It turned out that:
- I removed a lot of boilerplate code
- Templates change detection is optimized for signals
- I was working closer to Angular
- I could remove a dependency, which is always a nice thing to do if it makes sense

I refactored my entire codebase, (the private version of that repository contains a bunch of applications) and it was
a great success. My code got cleaned up a lot, performance was improved and I could work with **"just Angular"**

### Dropping ViewModels

ViewModels also don't make sense anymore when it comes to using signals.
When using signals I calculated my ViewModels like this:

```typescript
export class UserComponent {
    readonly #firstName = signal('Brecht');
    readonly #lastName = signal('Billiet');
    
    private readonly viewModel = computed(() => {
        return {
            fullName: `${this.#firstName()} ${this.#lastName()}`
        }
    })
    protected get vm () {
        return this.viewModel();
    }
}
```

In the template I could now read the vm like this:

```html
{%raw%}{{vm.fullName}}{%endraw%}
```

Why was I still doing this? What was I trying to solve?
- I didn't want template logic.
- I didn't want multiple subscriptions on my Observables.
- I didn't want multicasting issues.
- I didn't want to deal with `combineLatest` stuff or other RxJS complexity.

All those arguments were invalidated by signals. I could just use computed signals and optimise performance and remove boilerplate code:

```typescript
export class UserComponent {
    readonly #firstName = signal('Brecht');
    readonly #lastName = signal('Billiet');
    
    protected readonly fullName = computed(() => `${this.#firstName()} ${this.#lastName()}`);
}
```

The template would now look like this:

```html
{%raw%}{{fullName()}}{%endraw%}
```

While I didn't like the `()` syntax in the template, there was another important reason why I decided to drop viewModels.
Angular is working on more optimized change detection in templates, and having one big computed signal would just be bad
for performance.
In other words: **Ditching ViewModels is good for performance**

## Some best practices when it comes to state-management

### Keep the state as low as possible

If you can keep it in your component, keep it there. Do you need to share it with a sibling, keep it in your parent component.
If you only need it in your feature, keep it in your feature. Only if you really need it on a global level, make it a singleton.
Providing the state as low as possible will result in simpler state management and less complexity.

### Don't expose writable signals

Keep your writable signals private to ensure unidirectional dataflow.
You can expose setters or methods to update a piece of state. To expose a signal without its write functionalities
we can use the `asReadonly()` method.

```typescript
@Injectable()
export class ShoppingCartStateMachine {
    readonly #entries = signal<ShoppingCartEntry[]>([]);
    
    // Don't expose the writiable signal
    public readonly entries = this.#entries.asReadonly()
    
    // State update methods
    public addToCart(entry: ShoppingCartEntry) {...}
    public deleteFromCart(id: number): void {...}
    public updateAmount(id: number, amount: number): void {...}
}
```

### Use signals for state management, RxJS for event management

While we can ditch 90% of the RxJS functionality and simplify it with signals.
RxJS for state management is hard, because we have:
- Multicasting issues
- Glitches
- Error handling
- Memory leaks
- A good knowledge of RxJS

For state management we should use signals. For event management we should use RxJS.
Angular has created RxJS interoperability for that:
- `toSignal(obs$)` creates a signal from an observable
- `toObservable(signal)` creates an observable from a signal

When do we need it, when managing events, for instance while fetching data.
In the following example we want to load users based on a query, sorting and pageIndex.
It should be clear what is state management and what is event management:

```typescript
export class UsersListSmartComponent {
    ...
    // State, extracting from the activated route
    readonly #query = toSignal(
        this.activatedRoute.params.pipe(map(v => v['query'])), 
        { initialValue: this.activatedRoute.snapshot.params['query'] }
    );
    
    // Simple state
    readonly #sorting = signal({direction: 'ascending', sortBy: 'name'});

    // Simple state
    readonly #pageIndex = signal(0);

    // Simple state
    readonly #input = computed(() => ({
            query: this.#query(), 
            sorting: this.#sorting, 
            pageIndex: this.#pageIndex
    }));

    // Event management
    protected readonly results = toSignal(toObservable(this.#input).pipe(
        switchMap(v => this.userService.fetch(v.query, v.sorting, v.pageIndex))
    ), { initialValue: []})
}

```

## Conclusion

Angular state management changed a lot and signals made a lot of problems disappear.
It always pays of to zoom out and re-evaluate, even if it means not using your own libraries anymore.
ViewModels were a great pattern for RxJS but with signals they are not needed anymore and they even hurt performance.
I am working on gigantic codebases, even with complex state management, and those codebases keep on scaling and scaling.
The most important tip I can give you is: **KEEP IT SIMPLE AND STUPID**. If you don't **need** a third party library, don't use it.
If it makes you more productive... Great! Use it, but just know that you don't need it.
I hope you enjoyed this article! If you have questions, feel free to reach out!