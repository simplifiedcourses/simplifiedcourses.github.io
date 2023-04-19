---
layout: post
title:  "Angular ExpressionChangedAfterItHasBeenCheckedError: NG0100 Simplified and reverse-engineered"
date:   2022-12-09
published: true
comments: true
categories: [Angular, Change Detection]
cover: assets/ng0100.jpg
description: "ExpressionChangedAfterItHasBeenCheckedError: The error that makes Angular developers cringe. Let's reverse-engineer this error."
---

This might be one of the most googled Angular errors out there: The "Angular `ExpressionChangedAfterItHasBeenCheckedError` **Error: NG0100 expression has changed after it was checked.**". Every Angular developer that uses Angular
on a regular basis has probably encountered this error more than once already.
It could lead to frustration and some developers even blame Angular for this error, but the truth is:
It's Angular that is telling us that **we are doing something wrong**, and most of the time we are breaking unidirectional dataflow or doing side effects.

This error tells us that something has changed after Change Detection has already run, which is not what we want.
When the Change Detection Cycle is finished, we don't want our application to update any views anymore.

## When does this happen?

Let's solve the mystery, when does this error occur? It's simple: **If the template is updated when Change Detection has already finished**. That's it. 

**Note:**
In the V2 version of the [Angular Change Detection Simplified book](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"} we will reverse-engineer the Angular codebase together and show the internals of this error more in-depth than we do in this article.
Nevertheless, this should already be a **clear deep-dive** into the inner workings of Angular Change Detection in terms of its `ExpressionChangedAfterItHasBeenCheckedError` error.

### The Angular ExpressionChangedAfterItHasBeegnCheckedError behind the scenes

The `ExpressionChangedAfterItHasBeenCheckedError` error in Angular is a security mechanism that checks if the developer hasn't done something bad, like changing something in a wrong life cycle hook. In order to understand this a bit better, let's dive into the source code of Angular and see what happens behind the scenes:

In Angular, there is a [tick()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/application_ref.ts#L1001){:target="_blank"} function that triggers Change Detection.
In development mode, [tick()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/application_ref.ts#L1001){:target="_blank"} runs change detection **twice**:
- It runs Change Detection once by executing the [detectChanges()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/view_ref.ts#L273){:target="_blank"} function.
- It runs Change Detection a second time by executing the [checkNoChanges()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/view_ref.ts#L283) function. Below we find a simplified snippet of how the [tick()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/application_ref.ts#L1001){:target="_blank"} function works.

```typescript
tick(): void {
    for (let view of this._views) {
        // First time
        view.detectChanges();
    }
    if (ngDevMode) {
        for (let view of this._views) {
            // Second time (only in dev)
            view.checkNoChanges();
        }
    }
}
```

Both the [detectChanges()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/view_ref.ts#L273) function and the [checkNoChanges()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/view_ref.ts#L283){:target="_blank"} function trigger change detection. The first one runs with `isInCheckNoChangesMode` set to false.
The second one with `isInCheckNoChangesMode` set to true.
Every time a template is executed, the [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"} function will get executed for every binding in that template.
It will keep track of the value from the first Change Detection cycle ([detectChanges()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/view_ref.ts#L273){:target="_blank"}) in `lView[bindingIndex]`
and it will check the value in `lView[bindingIndex]` in the second Change Detection cycle [detectChanges()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/view_ref.ts#L283){:target="_blank"}. If those 2 values differ, Angular throws an error.

In other words, it will compare the value of `lView[bindingIndex]` between the first Change Detection cycle and the second Change Detection cycle **in development mode**.
If those values don't match... Guess what?! You got an `ExpressionChangedAfterItHasBeenCheckedError` thrown!


### How is a template normally handled?

To understand how and when this error is being thrown we have to understand how the template is being evaluated. 
The following snippet is an example of an `AppComponent` that renders a `{% raw %}{{name}}{% endraw %}` expression and a `HelloComponent` that doesn't do anything at the moment:

```typescript
@Component({
    ...
    template: `
    {% raw %}{{name}}{% endraw %}
    <hello></hello>
    `,
})
export class AppComponent {
    name = 'John';
}

@Component({...})
export class HelloComponent implements OnInit, AfterViewInit {

    public ngOnInit(): void {
    }

    public afterViewInit(): void {
    }
}

```

This will render **John** as a result of the `{% raw %}{{name}}{% endraw %}` expression and will handle the `HelloComponent` after that.

The template of this snippet will get executed in the following order (top to bottom).
First, `{% raw %}{{name}}{% endraw %}` will get taken care of, then `<hello></hello>` will get handled.

- Change Detection **first** runs for `AppComponent` with `isInCheckNoChangesMode` set to `false` (the initial execution triggered by [detectChanges()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/view_ref.ts#L273){:target="_blank"}):
    - `{% raw %}{{name}}{% endraw %}` is being rendered, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"}: `lView[bindingIndex]` is set to the current value: **John**
    - `HelloComponent` gets instantiated.
    - `HelloComponent` its `ngOnInit()` life cycle hook is executed.
    - Change Detection on `HelloComponent` is finished.
    - `HelloComponent` its `ngAfterViewInit()` life cycle hook is executed.
    - Change Detection on the `AppComponent` is finished.
    - `AppComponent` its `ngAfterViewInit()` life cycle hook is executed.
- Change Detection now runs **for the second time** on `AppComponent` with `isInCheckNoChangesMode` set to `true` this time (the second execution triggered by [detectChanges()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/view_ref.ts#L283){:target="_blank"}):
    - `{% raw %}{{name}}{% endraw %}` is being handled, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"}: compares current value with previous value on `lView[bindingIndex]` and throws `ExpressionChangedAfterItHasBeenCheckedError` if the value is different. The value would both be **John** in the previous snippet, so there is no error here.
    - Change Detection on `HelloComponent` is finished.
    - `HelloComponent` its `ngAfterViewInit()` life cycle hook is executed.
    - Change Detection on the `AppComponent` is finished.
    - `AppComponent` its `ngAfterViewInit()` life cycle hook is executed.

This is the basic flow and will not throw any errors so far.

## Triggering the ExpressionChangedAfterItHasBeenCheckedError

There are multiple ways to trigger this error and in this block, we will go over a few different scenarios:

### Updating the parent in the ngOnInit() hook of the child

The following code is not really a good practice, but let's do this for demo purposes anyway.
We will let `HelloComponent` update the `{% raw %}{{name}}{% endraw %}` property of the `AppComponent` from **John** to **Jane** in its `ngOnInit()` lifecycle hook:

```typescript
@Component({
    ...
    template: `
    {% raw %}{{name}}{% endraw %}
    <hello></hello>
    `,
})
export class AppComponent {
    name = 'John';
}

@Component({...})
export class HelloComponent implements OnInit {
    private readonly appComponent =  inject(AppComponent);

    public ngOnInit(): void {
        // ExpressionChangedAfterItHasBeenCheckedError
        this.appComponent.name = 'Jane';  
    }
}
```

This would change the name property to **Jane** after the `{% raw %}{{name}}{% endraw %}` was rendered and the `lView[bindingIndex]` was set to **John**. 
Let's take a look at the following flow:

- Change Detection runs on `AppComponent` with `isInCheckNoChangesMode` set to `false`:
    - `{% raw %}{{name}}{% endraw %}`  is being rendered, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"}: `lView[bindingIndex]` is set to the current value: **John**.
    - `HelloComponent` gets instantiated.
    - `HelloComponent` its `ngOnInit()` life cycle hook is executed and sets the `{% raw %}{{name}}{% endraw %}` property of `AppComponent` to **Jane**.
- Change Detection runs **for the second time** on `AppComponent` with `isInCheckNoChangesMode`set to `true` this time:
    - `{% raw %}{{name}}{% endraw %}`  is being rendered, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"}: compare current and old values on `lView[bindingIndex]`. **Boom!** This will throw `ExpressionChangedAfterItHasBeenCheckedError` because the value used to be **John** and now it is **Jane**.

We have clearly done something wrong... It does not make sense to set the value of a parent component in the `ngOnInit()` life cycle hook of a child component.
Here we broke the unidirectional dataflow and Angular lets us know that in development mode by throwing the `ExpressionChangedAfterItHasBeenCheckedError` error.

[Check the Stackblitz example](https://stackblitz.com/edit/angular-ivy-2vgmwq){:target="_blank"}

### Order matters when updating on ngOnInit()

When updating parent values on the `ngOnInit()` of a child component, it becomes even more dangerous because the order of the template of the parent component matters.
Here we see that we have reversed the order of the 2 lines in the template, which will result in a completely different outcome. The `HelloComponent` will get handled first and the `{% raw %}{{name}}{% endraw %}` expression will get handled after that:

```typescript
@Component({
    ...
    template: `
    <!-- Hello is executed first now -->
    <hello></hello>
    {% raw %}{{name}}{% endraw %}
    `,
})
export class AppComponent {
    name = 'John';
}

@Component({...})
export class HelloComponent implements OnInit {
    private readonly appComponent =  inject(AppComponent);

    public ngOnInit(): void {
        // No ExpressionChangedAfterItHasBeenCheckedError
        this.appComponent.name = 'Jane';  
    }
}
```

The `Hello` component will have executed its `ngOnInit()` function before the `{% raw %}{{name}}{% endraw %}` gets executed. This means that the moment the `lView[bindingIndex]` is set, the value would already be **Jane**, so in this case, the `ExpressionChangedAfterItHasBeenCheckedError` error **would not be thrown**.

Here is the flow:
- Change Detection runs on `AppComponent` with `isInCheckNoChangesMode` set to `false` (initial Change Detection cycle):
    - `HelloComponent` gets instantiated
    - `HelloComponent` its `ngOnInit()` life cycle hook is executed and sets the `{% raw %}{{name}}{% endraw %}` property of `AppComponent` to **Jane**.
    - `{% raw %}{{name}}{% endraw %}`  is being rendered, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"} => `lView[bindingIndex]` is set to the current value: which is **Jane** now (it's not **John** anymore).
- Change Detection runs **for the second time** on `AppComponent` with `isInCheckNoChangesMode` set to `true` this time (second Change Detection cycle):
    - `{% raw %}{{name}}{% endraw %}`  is being rendered, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"} => compare value on `lView[bindingIndex]`:
     this will throw **not** `ExpressionChangedAfterItHasBeenCheckedError` because the value was already **Jane** and it is still **Jane**.

[Check the Stackblitz example](https://stackblitz.com/edit/angular-ivy-wenrnn){:target="_blank"}

See how important it is to know how this works behind the scenes? The order of our templates determines if the `ExpressionChangedAfterItHasBeenCheckedError` error is thrown or not.

### ngAfterViewInit()

The name of this lifecycle hook says it all... The view is already initialized, meaning Change Detection has already run.
At this moment we don't want the template to change anymore.

This means that if we would update something in this life cycle hook, the second Change Detection cycle would definitely pick up a new value here resulting in the `ExpressionChangedAfterItHasBeenCheckedError` error.

```typescript
@Component({
    ...
    template: `
    <hello></hello>
    {% raw %}{{name}}{% endraw %}
    `,
})
export class AppComponent {
    name = 'John';
}

@Component({...})
export class HelloComponent implements AfterViewInit {
    private readonly appComponent =  inject(AppComponent)

    public ngAfterViewInit(): void {
        // ExpressionChangedAfterItHasBeenCheckedError
        this.appComponent.name = 'Jane';
    }
}
```

Let's go over the flow again:
- Change Detection runs on `AppComponent` with `isInCheckNoChangesMode` set to `false` (initial Change Detection cycle):
    - `HelloComponent` gets instantiated
    - `HelloComponent` its `ngOnInit()` life cycle hook is executed (but is not implemented)
    - `{% raw %}{{name}}{% endraw %}`  is being rendered, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"}: `lView[bindingIndex]` is set to the current value: **John**.
    - Change Detection is finished so the `ngAfterViewInit()` lifecycle hook is being triggered and sets the `{% raw %}{{name}}{% endraw %}` property of `AppComponent` to **Jane**.
- Change Detection runs **for the second time** on `AppComponent` in `isInCheckNoChangesMode` `true` this time (second Change Detection cycle):
    - `HelloComponent` will get re-evaluated:
    - `{% raw %}{{name}}{% endraw %}`  is being rendered, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"}: compare values on `lView[bindingIndex]`:
     this will throw `ExpressionChangedAfterItHasBeenCheckedError` because the value used to be **John** and now it is **Jane**!

[Check the Stackblitz example](https://stackblitz.com/edit/angular-ivy-vrhuyq){:target="_blank"}

Changing the template in the `ngAfterViewinit()` lifecycle hook is a no-no. This life cycle hook is literally just executed after Change Detection has finished.
 
### Random values

Another way to trigger the `ExpressionChangedAfterItHasBeenCheckedError` error is to render a random value. Think about generated unique ids, timestamps, etc.
In this example we will see how easy it is to reproduce the `ExpressionChangedAfterItHasBeenCheckedError` error by just making sure that the value bound in the DOM is different on both Change Detection cycles:

```typescript
@Component({
    ...
    template: `
    {% raw %}{{rand()}}{% endraw %}
    `,
})
export class AppComponent {
    public rand(): number {
        return Math.random();
    }
}
```

Let's go over the flow :
- Change Detection runs on `AppComponent` with `isInCheckNoChangesMode` set to `false` (initial cycle):
    -  `{% raw %}{{rand()}}{% endraw %}`  is being rendered, which calls [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"}: `lView[bindingIndex]` is set to a random value.
- Change Detection runs **for the second time** on `AppComponent` in `isInCheckNoChangesMode` `true` this time (second cycle):
    -  `{% raw %}{{rand()}}{% endraw %}` is being rendered to a new random value, which calls  [bindingUpdated()](https://github.com/angular/angular/blob/15.0.0/packages/core/src/render3/bindings.ts#L46){:target="_blank"}: compare values on `lView[bindingIndex]`:
     this will throw `ExpressionChangedAfterItHasBeenCheckedError` because both random values are different.

[Check the Stackblitz example](https://stackblitz.com/edit/angular-ivy-gzr9nh){:target="_blank"}

## Real-life use cases and solving the issue

Respecting unidirectional dataflow in Angular applications is important. It will help us optimize Change Detection, it will result in more predictable code and will avoid the Angular `ExpressionChangedAfterItHasBeenCheckedError` error. A child component should not be updating the parent component when it is just rendered.

So... If we want to fix this error, we should optimize the architecture. If it's impossible to do that, we could wrap the code triggering this error in a `setTimeout()` with a delay of `0` which will put this code in the next macro task queue of the javascript event loop, and trigger a new Change Detection cycle.

```typescript
public ngAfterViewInit(): void {
    setTimeout(() => {
        // code that would have triggered the 
        // ExpressionChangedAfterItHasBeenCheckedError error
    }, 0);
}
```

Let's look at some examples of when this "dirty" workaround would be acceptable.

### Infinite scroll

When we are implementing an infinite scroll where the scroll component (child) has to calculate how many rows to load based on its own size after it is rendered. When the component is fully rendered and the `ngAfterViewInit()` lifecycle hook is called, the child component wants to let the parent know how many rows to load which would probably result in a spinner being shown. That spinner would result in the `ExpressionChangedAfterItHasBeenCheckedError` error.

### Child component loading data

[Putting dialogs behind child routes](https://blog.brecht.io/routed-angular-dialogs/){:target="_blank"} could be seen as a best practice, since it gives you more control over the dialog and you can
use the native browser functionality to bookmark, navigate etc.
Let's say that we open a `user-detail` dialog that would be in charge of fetching a user based on the `userId` in the URL.
This component would be a child component of the `users` page and it would live below a child `router-outlet`.

```html
<h1>Users</h1>
<spinner *ngIf="loading"></spinner>
<table>...</table>
<router-outlet>
    <!-- detail is rendered in here -->
</router-outlet>
```
The `Users` component gets rendered, and after that the `UserDetail` component gets rendered and it will fetch a call resulting in
a `loading` property on the parent to become true (to show the spinner)
This means that in the first Change Detection cycle `loading` would be `false`, and in the second Change Detection cycle `loading`  would be `true`.
This would again result in the  `ExpressionChangedAfterItHasBeenCheckedError` error.

## Conclusion

We learned that Angular runs Change Detection twice in development mode:
- Once in `detectChanges()`
- Once in `checkNoChanges()`
- Updating the template when Change Detection already run, throws the `ExpressionChangedAfterItHasBeenCheckedError` error
- This error tells us we are breaking unidirectional data flow and should be seen as a feature in Angular.
- When updating the value of a parent component in a child component in the `ngOnInit()` lifecycle hook, it depends on the structure of the template
- When updating the value of a parent component in a child component in the `ngAfterViewInit()` lifecycle hook, Change Detection has already run and the error
will be thrown.

You can learn more about this subject and Change Detection in general in the [Angular Change Detection Simplified ebook](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}, where we will learn how Angular ticks and works under the hood. The `ExpressionChangedAfterItHasBeenCheckedError` error is demystified completely there.
[ebook](https://www.simplified.courses/angular-change-detection-simplified-e-book)!
[![Angular Change Detection ebook](/assets/angular-performant-drag-and-drop/ebook.png)](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}

If you liked the article, please leave a comment!