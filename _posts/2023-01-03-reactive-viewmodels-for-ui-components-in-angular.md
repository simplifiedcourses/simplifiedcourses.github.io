---
layout: post
title:  "Reactive ViewModels for UI components in Angular"
date:   2023-01-03
published: true
comments: true
categories: [Angular, RxJS, State management]
cover: assets/reactive-viewmodels-for-ui-components-in-angular.jpg
description: "This article shows you how to use reactive ViewModels in Angular in UI components to create clear reactive flows."

---

A while ago I wrote about [Smart components, UI components and sandbox facades in Angular](https://blog.simplified.courses/smart-components-ui-components-and-sandbox-facades-in-angular/){:target="_blank"}.
This article is about reactive ViewModels for UI components in Angular. We will see how this approach will result in cleaner templates, and more reactive code and will give us some extra possibilities regarding performance optimization.

## What's a ViewModel

A ViewModel is something that has a lot of different definitions and meanings. 
There are lots of different opinions about what a ViewModel should represent and which responsibility it has.
This article is not about starting a discussion and we will be clear about what it means for the context of this article.
**A reactive ViewModel is a reactive model that contains up-to-date data that is specific to the template of a component.**

**Update: We have updated the boilerplate and performance of ViewModels in this article:**
[Reactive input state for angular ViewModels](https://blog.simplified.courses/reactive-input-state-for-angular-viewmodels/){:target="_blank"}.

## The pager

To illustrate the power behind a reactive ViewModel for a UI component, we need an example.
For this article, we will create a reactive Angular pager component.

Why a pager? 
- It holds some complexity: It has to calculate some stuff based on the inputs
- It's standalone
- It's a UI component, meaning a dumb component. ViewModels can also be used in smart components but this article is about ViewModels in UI components.

Let's list the inputs of this **pager component**:

```typescript
/**
 * The amount of items a page should show
 */
@Input() public itemsPerPage: number;

/**
 * The total of items in the database
 */
@Input() public total: number;

/**
 * The index of the page (starts from 0)
 */
@Input() public pageIndex: number;
```

The template of our pager looks like this:

```html
  Showing {%raw%}{{ itemFrom }}{%endraw%} to {%raw%}{{ itemTo }}{%endraw%} of {%raw%}{{ total }}{%endraw%} entries
  <button (click)="goToStart()" [disabled]="previousDisabled">Begin</button>
  <button (click)="previous()" [disabled]="previousDisabled">Previous</button>
  <button (click)="next()" [disabled]="nextDisabled">Next</button>
  <button (click)="goToEnd()" [disabled]="nextDisabled">End</button>
```

We can see that we need 5 values in our template:
- `itemFrom`
- `itemTo` 
- `total`
- `previousDisabled`
- `nextDisabled`

The `total` property is the same as its input so we don't have to calculate this one, but the other 4 values need to be calculated.
We could calculate this in the template, but we want to put this in the class because:
- We don't want to repeat the logic for `previousDisabled` and `nextDisabled`.
- We want to avoid logic in the templates because it is harder to unit test
- Less complexity in the template results in better separation of concerns

Calculating this in the class means that we have to care about the order of when inputs get new values.
The inputs: `itemsPerPage`, `total` and `pageIndex` can change all the time and when one of these changes, all the values (except `total`) need to be recalculated.
We could do that in the `ngOnChanges()` lifecycle hook but that would result in brittle code and in this article we want to use a **reactive approach**.

### Creating a reactive ViewModel

We will use a combination of **input setters** and **BehaviorSubjects** to achieve this.
A **BehaviorSubject** is a **ReplaySubject** that replays the last value and has an initial value.
Why do we need an initial value? Because later on we will use `combineLatest` to create a reactive ViewModel object and this operator will not emit unless all its source observables have emitted (which is exactly what a **BehaviorSubject** does initially).
We will suffix the subjects with `$$` so we know they are not regular observables but **subjects**:

```typescript
// Creating the BehaviorSubjects with an initial value
private readonly itemsPerPage$$ = new BehaviorSubject<number>(0);
private readonly total$$ = new BehaviorSubject<number>(0);
private readonly pageIndex$$ = new BehaviorSubject<number>(0);
```

Now let's update the **Inputs** with setters that will feed the **BehaviorSubjects** we have just created:

```typescript
/**
 * The amount of items a page should show
 */
@Input() public set itemsPerPage(v: number) {
    this.itemsPerPage$$.next(v);
};

/**
 * The total of items in the database
 */
@Input() public set total(v: number) {
    this.total$$.next(v);
};

/**
 * The index of the page (starts from 0)
 */
@Input() public set pageIndex(v: number) {
    this.pageIndex$$.next(v);
};
```

The next step is to create the ViewModel. We can start with the `ViewModel` type, which holds the properties that our template needs:
- `itemFrom`
- `itemTo` 
- `total`
- `previousDisabled`
- `nextDisabled`
  
We can add this type in the same file as our component since it won't be used anywhere else:

```typescript
type ViewModel = {
    itemFrom: number;
    itemTo: number;
    total: number;
    previousDisabled: boolean;
    nextDisabled: boolean;
}
```

This feels nice: **a specific type just for our template**. This means our template should not access anything else unless some public functions that our component class offers.

Now let's create a `vm$` observable that will get updated every time one of our **inputs** changes:

```typescript
public readonly vm$: Observable<ViewModel> = combineLatest({
    itemsPerPage: this.itemsPerPage$$,
    total: this.total$$,
    pageIndex: this.pageIndex$$,
  }).pipe(
    map(({ itemsPerPage, total, pageIndex }) => {
      // we could extract this in a reusable function if
      // we want to.
      return {
        total,
        previousDisabled: pageIndex === 0,
        nextDisabled: pageIndex >= Math.ceil(total / itemsPerPage) - 1,
        itemFrom: pageIndex * itemsPerPage + 1,
        itemTo:
          pageIndex < Math.ceil(total / itemsPerPage) - 1
            ? pageIndex * itemsPerPage + itemsPerPage
            : total,
      };
    })
  );
```

Let's not focus on the complexity inside the ViewModel, but it looks better here than it would be in the template, right?!
Did we mention that you only need one async pipe for this, meaning only one subscription?
We can use `ng-container` in combination with `*ngIf` to only subscribe to our ViewModel once and create a local template variable:

```html
<ng-container *ngIf="vm$|async as vm">
    Showing {%raw%}{{ vm.itemFrom }}{%endraw%}
    to {%raw%}{{ vm.itemTo }}{%endraw%} of 
    {%raw%}{{ vm.total }}{%endraw%} entries
    <button (click)="goToStart()" 
      [disabled]="vm.previousDisabled">
      Begin
    </button>
    <button (click)="previous()" 
      [disabled]="vm.previousDisabled">
      Previous
    </button>
    <button (click)="next()" 
      [disabled]="vm.nextDisabled">
      Next
    </button>
    <button (click)="goToEnd()" 
      [disabled]="vm.nextDisabled">
      End
    </button>
</ng-container>
```

Now we see that the `vm` is available everywhere as a template variable which makes it convenient to consume the values in the template. There is one more problem to solve. There is no interaction with the buttons yet. We need one **Output** and four methods to make our pager complete:

```typescript
/**
 * Notifies the parent when the page has changed
 */
@Output() public readonly pageIndexChange = new EventEmitter<number>();

public goToStart(): void {
    this.pageIndexChange.emit(0);
}

public next(vm: ViewModel): void {
    this.pageIndexChange.emit(/*pageIndex*/ + 1);
}

public previous(vm: ViewModel): void {
    this.pageIndexChange.emit(/*pageIndex*/ - 1);
}

public goToEnd(vm: ViewModel): void {
    this.pageIndexChange.emit(Math.ceil(vm.total / /*itemsPerPage*/) - 1);
}
```

We can see here that we are missing 2 vital properties to communicate with the parent component:
- `pageIndex`
- `itemsPerPage`

Let's add these to the ViewModel type and update the ViewModel like this:

```typescript
type ViewModel = {
  itemFrom: number;
  itemTo: number;
  total: number;
  previousDisabled: boolean;
  nextDisabled: boolean;
  pageIndex: number;
  itemsPerPage: number;
};
...
public readonly vm$: Observable<ViewModel> = combineLatest(...)
  .pipe(
    map(({ itemsPerPage, total, pageIndex }) => {
      return {
        // add them here so they are available on the ViewModel
        itemsPerPage,
        pageIndex,
        ...
      };
    })
  );
...
```

This technique is powerful, since we have the `vm` as a template variable we have access to the latest value of the ViewModel everywhere.
We can now pass the `vm` property to the `next`, `previous` and `goToEnd` functions where we need the `pageIndex` and `itemsPerPage`:

```typescript
...
@Component({
  ...
  template: `
    ...
    <button (click)="previous(vm)" ...>
      Previous
    </button>
    <button (click)="next(vm)" ...>
      Next
    </button>
    <button (click)="goToEnd(vm)" ...>
      End
    </button>
    ...
  `,
})
export class PagerComponent {
  ...
  // pass the vm
  public next(vm: ViewModel): void {
    this.pageIndexChange.emit(vm.pageIndex + 1);
  }

  // pass the vm
  public previous(vm: ViewModel): void {
    this.pageIndexChange.emit(vm.pageIndex - 1);
  }

  // pass the vm
  public goToEnd(vm: ViewModel): void {
    this.pageIndexChange.emit(Math.ceil(vm.total / vm.itemsPerPage) - 1);
  }
}

```

We now have a fully working pager component that is reactive, testable and easy to read.

## Optimizing for performance

The calculations are not that heavy but we could still optimize the ViewModel a bit more by leveraging the `distinctUntilChanged` operator. This operator will only emit new results when the new value is different than the previous value:

```typescript
 public readonly vm$: Observable<ViewModel> = combineLatest({
    itemsPerPage: this.itemsPerPage$$.pipe(distinctUntilChanged()),
    total: this.total$$.pipe(distinctUntilChanged()),
    pageIndex: this.pageIndex$$.pipe(distinctUntilChanged()),
  }).pipe(
    ...
  );
```

For this example, the performance gain will be trivial, but it should prove the amount of control we have on how to optimize
this for performance.

## Demo

You can check out the full working component in this [Stackblitz example](https://stackblitz.com/edit/angular-ivy-sk28wg?file=src%2Fapp%2Fapp.component.ts){:target="_blank"}

## Conclusion

We have learned that introducing ViewModels on UI components can be done by using:
- **BehaviorSubjects**
- **Setters**
- the `combineLatest` operator

Introducing these ViewModels results in:
- Cleaner templates
- Better testability
- Less redundancy of complexity in templates
- A way to optimize for performance, eg for [Change Detection](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}
- A more reactive way of programming

If you liked the article, please leave a comment! Creating ViewModels could be done a lot cleaner by using [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"}

If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training) and [Angular Coaching](https://www.simplified.courses/angular-coaching)