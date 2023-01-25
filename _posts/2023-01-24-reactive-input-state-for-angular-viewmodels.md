---
layout: post
title:  "Reactive Input state for Angular ViewModels"
date:   2023-01-24
published: true
comments: true
categories: Angular, RxJS
cover: assets/reactive-input-state-for-viewmodels-in-angular.jpg
description: "This article shows how we can optimize the number of emissions and calculations when it comes to calculating ViewModels from input state in Angular"

---

## Explaining the problem

In a previous article, we have written about [Reactive ViewModels for Ui components in Angular](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"}. If you haven't read this article yet, we recommend you read it first, since this article is an improvement on the other one. With a ViewModel, we mean a **Reactive model specifically created for the view (template)**. In short, it is an RxJS Observable that contains all the properties that our template needs to render correctly. It contains all the properties and only those properties.

Some of the advantages are:
- Only one async pipe is needed.
- Only one subscription is created.
- No more [null issues with the async pipe](https://stackoverflow.com/questions/61780339/angular-ivy-stricttemplates-true-type-boolean-null-is-not-assignable-to-type){:target="_blank"}.
- Clear separation of concerns.
- We can move logic from the template to the ViewModel of the class instance.

Here we can see some short code for a ViewModel of a pager component:
(Pay attention to the single subscription in the `*ngIf` directive)

```html
<ng-container *ngIf="vm$|async as vm">
    Showing {%raw%}{{ vm.itemFrom }}{%endraw%}
    to {%raw%}{{ vm.itemTo }}{%endraw%} of 
    {%raw%}{{ vm.total }}{%endraw%} entries
    ...
</ng-container>
```

In the article mentioned before, we used a combination of setters and **BehaviorSubjects** to create observables from our **@Input()** properties. We then used the `combineLatest` operator to create our ViewModel and used the `distinctUntilChanged` operator to optimize it for performance.
While this can be seen as a nice approach, there are still a few problems with this way of creating ViewModels:

- Setters of inputs can be called multiple times in one [Change Detection](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"} cycle.
  - This can result in multiple emissions of the `combineLatest` operator in one Change Detection cycle.
  - This can result in multiple calculations of the ViewModel in one Change Detection cycle.
  - This will result in multiple `markForCheck()` executions in one Change Detection cycle due to the `async` pipe, even though this is a trivial performance loss.
- Creating a setter and a BehaviorSubject for every **@Input()** property results in boilerplate code.
- Manual `combineLatest()` with `distinctUntilChanged` results in boilerplate and RxJS complexity. This could be automated.

The fact that the setters can be called multiple times per Change Detection cycle could be fixed by using a `debounceTime(0)` statement:
```typescript
public readonly vm$: Observable<ViewModel> = combineLatest({...}).pipe(
  debounceTime(0), // Combine events that happen at the same time
  map(({ itemsPerPage, total, pageIndex }) => {
    ...
  })
);
```

This approach also has its drawbacks since it results in:
- Unnecessary boilerplate code
- A new Change Detection cycle is being triggered. (added to the macrotask queue of the javascript event loop)
- It's a hack...

## The goal of this article 

In this article, we will create **one** observable that is populated with the values of the **@Input()** properties. We will continue on the example of the previous article which is a pager component.
What we want to achieve is the following:
- The Observable should emit even if there are Inputs that are not being set (Optional inputs).
- The Observable should **emit only once per Change Detection cycle**.
- The Observable should emit initially with the initial values
- It should work with default values of **@Input()** properties
- The Observable should emit after initialization only if one of the inputs has changed.
- We want to remove as much boilerplate code as possible

In this article, we will cover the creation of an **InputStateModel** and we will see how we can remove boilerplate by using **Typescript Decorators** in a second version. This is the first article of a series of multiple articles: In the next article, we will dive into the component state and later on we will see how we can combine the Input state, the component state and ViewModels in Angular. 

## The ngOnChanges life cycle hook

Let's focus on passing the input state into our **InputStateModel** which would hold one observable that is emitted for every **@Input()** property change.
The default way to know if any **@Input()** properties have gotten new data in Angular would be the use of the `ngOnChanges` lifecycle hook. This lifecycle hook gets a `SimpleChanges` parameter passed that contains the current changes and the previous changes for every input. The beautiful thing about this hook is that it only **gets executed once per Change Detection cycle**.
Using this lifecycle hook can get brittle because it is not typed. Let's create our own `TypedSimpleChanges` interface that takes a generic type to fix this.
Let's add a `PagerInputState` type as well to complete it.

```typescript
// Interface for typed simple changes
export interface TypedSimpleChanges<T> extends SimpleChanges {}

// Type for the Observable that will hold the input state
type PagerInputState = {
  itemsPerPage: number;
  total: number;
  pageIndex: number;
}

export class PagerComponent implements OnChanges {
  ...
  public ngOnChanges(changes: TypedSimpleChanges<PagerInputState>): void {}
}
```

In the `ngOnChanges` life cycle hook we want to feed some kind of Observable with data that we can consume or that we can generate a ViewModel from.

We want to create an `InputStateModel` class that is an injectable that is provided on `PagerComponent`. We want to inject it into the pager component but we also want it to be destroyed when the `PagerComponent` gets destroyed. When we provide the `InputStateModel`in the `providers` as shown below, the `ngOnDestroy` lifecycle hook of `InputStateModel` will get executed when `PagerComponent` gets destroyed. In other words: The instance of `InputStateModel` will get destroyed when the instance of `PagerComponent` gets destroyed. We can also see that we use the `ngOnChanges` life cycle hook to pass the simple changes to the `update()` function of `this.inputStateModel`.

```typescript
@Component({…
  // Create new instance of InputStateModel
  // That is tied to the instance of PagerComponent
  providers: [InputStateModel]
})
export class PagerComponent implements OnChanges {
  // Inject the InputStateModel of type PagerInputState
  private readonly inputStateModel = inject(InputStateModel<PagerInputState>);
  // Extract the state from the model
  private readonly inputState$ = this.inputStateModel.state$;

  @Input() public itemsPerPage: number = 0;
  @Input() public total: number = 0;
  @Input() public pageIndex: number = 0;

  // Update the type to TypedSimpleChanges that is generic
  public ngOnChanges(changes: TypedSimpleChanges<PagerInputState>): void {
    // Pass the changes to the inputStateModel that will handle everything.
    this.inputStateModel.update(changes);
  }

  // generate ViewModel from the inputState$
  public readonly vm$: Observable<ViewModel> = this.inputState$.pipe(
    map(({ itemsPerPage, total, pageIndex }) => {...})
  );
}
```

The code above is the first version of how we can consume the `InputStateModel`. We still have to implement the `InputStateModel` later, but we can see that we have reduced the boilerplate code a lot:
- There are no more BehaviorSubjects.
- There ar no more setters.
- There are no more `combineLatest()` nor `distinctUntilChanged()` operators to be seen.
- We can just use the `map()` operator to create the `vm$` ViewModel from `this.inputState$`.

**Most importantly: It will only be called once per Change Detection cycle**.

## Implementing InputStateModel

Before we start implementing this **Injectable**, let's list the features of this class:
- It should create a BehaviorSubject that holds the input state when the `update` function is called for the first time.
- It should update that BehaviorSubject when the `update` function is called afterward.
- It should cancel the subscriptions on the `ngOnDestroy` life cycle hook.
- It should only emit new values when one or more **@Input()** properties have changed.

Let's dive in, the explanation of the code is added in the comments:

```typescript
@Injectable()
export class InputStateModel<T> implements OnDestroy {
  // This injectable will get destroyed when the component who
  // provides this instance is destroyed, We will use the takeUntil operator
  // to cleanup every subscription to state$
  private readonly destroy$$ = new Subject<void>();

  // The update function will create the state$$ BehaviorSubject
  // We will use this initialized$$ BehaviorSubject to only expose state
  // when this instance is initialized
  private readonly initialized$$ = new BehaviorSubject<boolean>(false);

  // This will be created and nexted in the update function
  // This holds the value of all the @Input() properties
  private state$$: BehaviorSubject<T>;

  // Expose a state$ observable when this instance is initialized
  public readonly state$: Observable<T> = this.initialized$$.pipe(
    filter(v => !!v), // Only expose the state when initialized
    switchMap(() => {
      // This is here to avoid Typescript compilation issues
      if(!this.state$$){
        throw new Error('State must be initialized. Did you forgot to call the connect method?')
      }
      return this.state$$
    }),
    // Only emits when one or more of the @Input() properties change 
    distinctUntilChanged((previous: T, current: T) => {
      const keys = Object.keys(current);
      return keys.every(key => {
        return current[key] === previous[key]
      })
    }),
    // Clean up after ngOnDestroy
    takeUntil(this.destroy$$),
  )
  
  // Next the destroy$$ subject when this instance gets destroyed
  // This is used to avoid memory leaks
  public ngOnDestroy(): void {
    this.destroy$$.next();
  }

  /**
   * Will be called from within the ngOnChanges
   * life cycle hook
   **/
  public update(changes: TypedSimpleChanges<T>):void {
   const keys = Object.keys(changes);
    // If the state$$ BehaviorSubject is not created yet
    // (initial ngOnChanges):
    // Create a state for all @Input() properties
    // Create a new BehaviorSubject with that state,
    // and set the initialized$$ to true
    if (!this.state$$) {
      const state: T = {} as T;
      keys.forEach((key) => {
        state[key as keyof T] = changes[key].currentValue;
      });
      this.state$$ = new BehaviorSubject<T>(state);
      this.initialized$$.next(true);
    // If the state already exists:
    // Take the current state and only update the state with inputs
    // if the current value is different from the previous value
    } else {
      const state: T = { ...this.state$$?.value } as T;
      keys.forEach((key) => {
        if (changes[key].currentValue !== changes[key].previousValue) {
          state[key as keyof T] = changes[key].currentValue;
        }
      });
      // Only create one event for all @Input() properties
      this.state$$.next(state);
    }
  }
}
```

That's it. We can find a working [Stackblitz example of this code here](https://stackblitz.com/edit/angular-ivy-w9bmdk?file=src%2Fapp%2Fpager%2Fpager.component.ts){:target="_blank"}.

## Optimizing with a Typescript Decorator

The current simplified version of the PagerComponent still has some boilerplate code we would like to reduce:

```typescript
@Component({…
  providers: [InputStateModel] // Boilerplate
})
export class PagerComponent implements OnChanges {
  // Boilerplate
  private readonly inputStateModel = inject(InputStateModel<PagerInputState>);
  private readonly inputState$ = this.inputStateModel.state$;

  // Boilerplate
  public ngOnChanges(changes: TypedSimpleChanges<PagerInputState>): void {
    this.inputStateModel.update(changes);
  }
}
```

- We have to provide the `InputStateModel` in the `providers` property of `@Component`.
- We have to inject the `inputStateModel` and then extract the `state$` property from it.
- We have to implement `ngOnChanges` and update `this.inputStateModel` in there with the latest changes.
- This implementation does not work with default values of **@Input()** properties.

We would really love to reduce this to a one-liner that listens to `ngOnChanges`, `ngOnDestroy` but would still let us implement it in the pager component if we wanted to.
We could create an `@InputState()` decorator that does this for us and we would like to use it like this:

```typescript
@Component(...)
export class PagerComponent  {
  @Input() public itemsPerPage: number = 0;
  @Input() public total: number = 0;
  @Input() public pageIndex: number = 0;

  @InputState() private readonly inputState$!: Observable<PagerInputState>;
}
```
You can see the `!` syntax that will tell us that this property is always initialized because it would be the responsibility of the `@InputState()` decorator to initialize that observable.
This Decorator would use a variant of the `InputStateModel` to achieve everything we have accomplished before.

When using Typescript property decorators it is important to realize that this decorator applies to the prototype of the class and not to the instance that is created of that class.

Let's create an `input-state-decorator.ts` file and expose the `InputStateModel`:

```typescript
export function InputState<T>() {
  return function (
    target: any,
    key: string
  ) {
    // This secretModel will actually be kept as a property on the instance of
    // the component that uses this InputState decorator
    const secretInputModel = `secret${key}Model`;
    // This accessor is used to get access to the secretInputModel
    // This property will not exist on the instance of the component
    // that uses this InputState decorator
    const accessorInputModel = `accessor${key}Model`;

    // We need to keep InputStateModel on the instance
    // Since this decorator is static we need to use this syntax
    // to get access to this
    Object.defineProperty(target, accessorInputModel, {
      get: function () {
        // If it doesn't exist yet, create the InputStateModel
        if (!this[secretInputModel]) {
          this[secretInputModel] = new InputStateModel();
        }
        // return the InputStateModel
        return this[secretInputModel];
      },
    });

    // This is what the decorator will return 
    // (the actual input state of the InputStateModel)
    return {
      get: function () {
        return this[accessorInputModel].state$;
      }
    };
  };
}
```

Getting access to `this` in a Typescript property decorator seems a bit tricky and since this is not a deep dive into Typescript decorators we suggest having a look at the documentation if you would like to know more. We are always open to questions, just leave them in the comments.
In short: This decorator will keep an instance of `InputStateModel` on the instance of the component where the `InputState` decorator is used and will return the `state$` property of that `InputStateModel` instance to the property where this decorator is used on.

Now let's handle the `ngOnChanges` and `ngOnDestroy` logic. We will use the `ngOnChanges` life cycle hook to populate the `InputStateModel` and `ngOnDestroy` to clean it up:

```typescript
// Keep track of the original 2 lifecycle hooks
const origNgOnChanges = target.constructor.prototype.ngOnChanges;
const origNgOnDestroy = target.constructor.prototype.ngOnDestroy;

// overwrite the original ngOnChanges life cycle hook
target.ngOnChanges = function (simpleChanges: TypedSimpleChanges<T>): void{
  // if ngOnChanges is implemented execute it as well
  if (origNgOnChanges) {
    origNgOnChanges.apply(this, [simpleChanges]);
  }
  this[accessorInputModel].update(simpleChanges); // send changes to model
};

// Overwrite the original ngOnDestroy life cycle hook
target.ngOnDestroy = function (): void {
  // If ngOnDestroy is implemented execute it too
  if (origNgOnDestroy) {
    origNgOnDestroy.apply(this, []);
  }

  // Clean up the instance InputStateModel
  this[accessorInputModel].ngOnDestroy();
};
```


This will work but we have one more problem: This will not work with the default values of the inputs, since the `ngOnChanges` does not contain the default values of our **@Input()** properties.


We will have to update the implementation of the `ngOnChanges` method like this:

```typescript
target.ngOnChanges = function (simpleChanges: TypedSimpleChanges<T>): void{
  // get the keys of all the Input() properties
  const inputKeys = Object.keys(target.constructor.propDecorators).filter(
    (key) =>
      target.constructor.propDecorators[key].find(
        (decorator: any) =>
          decorator.type.prototype.ngMetadataName === 'Input'
      )
  );

  // Make a copy of the current SimpleChanges
  const simpleChangesToPass: TypedSimpleChanges<T> = { ...simpleChanges };

  // If there are inputs that are not part of the simpleChanges
  // it means they are default
  inputKeys
    .filter((inputKey) => !simpleChanges[inputKey])
    .forEach((inputKey) => {
      simpleChangesToPass[inputKey] = new SimpleChange(
        this[inputKey],
        this[inputKey],
        true
      );
    });

  this[accessorModel].update(simpleChangesToPass); // send changes to model

  // if ngOnChanges is implemented execute it as well
  if (origNgOnChanges) {
    origNgOnChanges.apply(this, [simpleChanges]);
  }
};
```

The decorator is finished and now we can create a ViewModel from our inputs without any boilerplate code:

```typescript
export class PagerComponent {
  // Create type safe observable that is updated and cleaned up automatically 
  @InputState() private readonly inputState$!: Observable<PagerInputState>;
  
  @Input() public itemsPerPage: number = 0;
  @Input() public total: number = 0;
  @Input() public pageIndex: number = 0;

  // Use the inputState$ to calculate a ViewModel
  public readonly vm$: Observable<ViewModel> = this.inputState$.pipe(
    map(({ itemsPerPage, total, pageIndex }) => {
      return {... };
    })
  );
  ...
}
```

You can check a fully working example in [this stackblitz](https://stackblitz.com/edit/angular-ivy-by6whw?file=src%2Fapp%2Fpager%2Fpager.component.ts){:target="_blank"}.


## Why did we write this ourselves?

- We want to update to newer Angular versions without any trouble
- We can add features ourselves, like snapshots for instance
- It's quite simple and we have full control over the code
- It's not that much code
- We are not diverging from the Angular ecosystem
- We have learned a bunch
- We are in charge of our own quality

## Conclusion

- We have reduced the boilerplate code a lot.
- We have hidden some RxJS complexities for the developer.
- We now have an optimized **@Input()** properties state observable that we can easily use to create a ViewModel.
- We don't have unwanted emissions or calculations.
- A Typescript property decorator is static, and we have to use a specific syntax to get a hold of `this`.
- Subscriptions are handled on `ngOnDestroy` and cleaned up automatically.
- We can still implement the `ngOnChanges` and `ngOnDestroy` life cycle hooks if we want to but we don't have to.
- We don't rely on any open-source projects and are working close to the Angular ecosystem

There are still a few problems though.
In the next article, we will fix the following problems:
- Our ViewModel still contains properties our template doesn't need: (`pageIndex` and `itemsPerPage`)
- If we set the **@Input()** properties ourselves, the state will not update
- The observable gets a new event every time an input property changes (in the same Change Detection cycle of course)
- How can we make this work together with other pieces of state.

This is the first step towards truly reactive components. In the next articles, we will dive deeper and use this principle as a cog in our reactive component state. If you liked the article, please leave a comment!