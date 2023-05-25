---
layout: post
title:  "The danger of using Angular HostListeners"
date:   2023-04-24
published: true
comments: true
categories: [Angular, Change Detection, Best Practices]
cover: assets/the-danger-of-using-angular-host-listeners.jpg
description: "Why are Angular HostListeners dangerous? In which scenarios would they over-trigger Change Detection?"
---

## What are Angular HostListeners?

Angular HostListeners are decorators that we can use to attach an event listener to a certain component. In this example, we create an event listener where we listen to any **click** event happening on a `HelloComponent`.

```typescript
@Component({
    ...
})
export class HelloComponent  {
    @HostListener('click', ['$event']) 
    public clicked(e: MouseEvent): void {
        console.log('Host got clicked', e);
    }
}
```

In the previous example when we would click on this `HelloComponent` the console would log something like **Host got clicked, PointerEvent{isTrusted: true}**

Usually, when we want to add click events we would use `@Output()` events for that but in some cases, we want to listen to events on the host.

Just like an `@Output()` event, Angular will mark the view of `HelloComponent` and all its parents as `lViewFlags.dirty` and because the `addEventListener` function is created in the inner zone, Change Detection would get triggered.
If you don't know how Change Detection in Angular works. Check out this [Angular Change Detection explained in 5 minutes video](https://www.youtube.com/watch?v=eNuMUslF8Bw&t=47s){:target="_blank"} or this [Angular Change Detection Simplified Ebook](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}

## When does this become dangerous?

While there is nothing wrong with HostListeners, it can become dangerous in terms of performance in some cases.
As the first argument of the `@HostListener` decorator, we can pass different strings to tell Angular what event it should listen to and where it should listen on.
The latter could be dangerous.

The argument passed could be one of the following:
- `@HostListener('click')`: Listens to the click of this component.
- `@HostListener('window:click')`: Listens to any click on the `window` object.
- `@HostListener('window:keydown')`: Listens to any key-down event on the `window` object.
- `@HostListener('window:keydown.enter')`: listens to only the **enter** key-down events on the `window` object.

The problem now becomes that we are not listening to an event on the actual host but we can listen to an event on `window` or `document`. Now this becomes dangerous.

There are scenarios where we want to listen to changes on the `window` or `document`.

### When we click outside an element

Think about when you have a dialog open and you want to close it when you click outside of that dialog. This HostListener could help you to close something when you click outside of that element:

```typescript
@Component({
    ...
})
export class DialogComponent  {
    @HostListener('document:click', ['$event.target'])
    public onClick(target: any) {
        const clickedInside = this.elementRef.nativeElement.contains(target);
        if (!clickedInside) {
            this.close.emit();
        }
    }
}
```

**Beware: Now Change Detection will trigger every time you click anywhere in your entire application as long as this component lives**. 

### More dangerous scenario

Think about having 20 popovers in your application that can be expanded and every one of them has a `@HostListener` on `document:click` that collapses it back again when you click somewhere else:

```typescript
@Component({
    ...
})
export class PopoverComponent  {
    @HostListener('document:click', ['$event.target'])
    public onClick(target: any) {
        const clickedInside = this.elementRef.nativeElement.contains(target);
        if (!clickedInside) {
            this.closed = true;
        }
    }
}
```

This now **will trigger Change Detection 20 times on every click**. Starts to become more dangerous isn't it?!


### When a window resizes or the user scrolls

When we want to get notified when the window resizes or the user scrolls is also a scenario why we would use HostListeners. An infinite scroll might be a good example for this one.

```typescript
@Component({
    ...
})
export class InfiniteScrollComponent  {
    @HostListener('window:resize', ['$event']) 
    public resized(e: MouseEvent): void {
        this.calculatePageOnResize(e);
    }
    @HostListener('document:scroll', ['$event']) 
    public scrolled(e: MouseEvent): void {
        this.calculatePageOnScroll(e);
    }
}
```

By having access to the `window:resize` and `document:scroll` events we get access to the information we needed to calculate the next page we need to load.

But... now on every **resize** and **scroll** event that is captured in our `InfiniteScrollComponent` Change Detection will get triggered multiple times per second **application wide...**
This can definitely result in slow applications. Most of the time it is not about how many components are detected for changes, but how many times Change Detection actually got triggered.

## The solution

While HostListeners can be a valid solution for our problems, it's easy to trigger `zone.js` and thus Change Detection all the time...
To solve this issue for the `document:scroll` event we could leverage the combination of the **outer zone** and the **inner zone** that angular has to offer us.
Again, if you are unfamiliar with Angular Change Detection, just watch the [5 minute video](https://www.youtube.com/watch?v=eNuMUslF8Bw&t=47s){:target="_blank"}

In this example, we will subscribe to the `document:scroll` event in the **outer zone** by running our code in `ngZone.runOutsideAngular()`so that Angular Change Detection would not be triggered, and when we do need to recalculate the page we would trigger change detection ourselves by using the `ngZone.run()` method:

```typescript
@Component({
    ...
})
export class InfiniteScrollComponent implements OnDestroy {
    private readonly ngZone = inject(NgZone);
    private readonly destroy$$ = new Subject<void>();
    constructor() {
        this.ngZone.runOutsideAngular(() => {
            fromEvent(document, 'scroll')
                .pipe(takeUntil(this.destroy$$))
                .subscribe((e: MouseEvent) => {
                    const newPage = this.calculatePage(e)
                    if(this.pageNeedsToBeChanged()){
                        this.ngZone.run(() => {
                        this.pageIndexChange(newPage)
                    });
                }
            });
        });

        // TODO: do the same for the resize event
    }
    public ngOnDestroy(): void {
        this.destroy$$.next();
    }
}
```

## Conclusion

What did we learn? We have learned that it is okay to use HostListeners in Angular but it is important to know that when we subscribe to events on the `window` or `document` it could have a significant impact on how many times Change Detection would get triggered **application wide** which could result in performance issues.

In these articles, we also optimize Change Detection by leveraging the **inner zone** and the **outer zone**:
- [CHATGPT helped me write a snake game in angular and how I optimized it for performance](https://blog.simplified.courses/how-chatgpt-helped-me-write-a-snake-game-in-angular-and-how-i-optimized-it-for-performance/){:target="_blank"}
- [Running Outputs outside zone.js for Angular performance Optimization](https://blog.simplified.courses/running-outputs-outside-zonejs-for-angular-performance-optimization/){:target="_blank"}
- [Angular performant drag-and-drop with RxJS](https://blog.simplified.courses/angular-performant-drag-and-drop-with-rxjs/){:target="_blank"}

If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training) and [Angular Coaching](https://www.simplified.courses/angular-coaching)