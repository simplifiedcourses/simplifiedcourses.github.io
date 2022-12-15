---
layout: post
title:  "Angular performant drag-and-drop with RxJS"
date:   2022-11-25
published: true
comments: true
categories: Angular 
cover: assets/angular-performant-drag-and-drop.png

---

Implementing drag-and-drop functionality can easily be achieved by leveraging one of the many
drag-and-drop libraries out there. We prefer to use the [Angular CDK](https://material.angular.io/cdk/drag-drop/overview){:target="_blank"}. occasionally, but sometimes we need something more customized.

In this article, we will learn how we can leverage RxJS in combination with Angular to implement drag-and-drop functionality that is optimized for performance.
As this concept might be seen as something complex, we will simplify it for you!

## The draggable box: Getting access to the native element

The drag-and-drop functionality we will write is showcased in the following Stackblitz:

<iframe src="https://angular-ivy-heyrwg.stackblitz.io/" width="100%" height="500" style="border:none">
</iframe>

The first thing that we want to do is create a simple box that we will drag later.
We will call this component `draggable-box` and for this article, the only thing we want to do is drag
it across our screen. For that, we need a reference to the `ElementRef` of the component.

We can achieve this by injecting the `ElementRef` and implementing the `AfterViewInit` interface.
When the `ngAfterViewInit()` lifecycle hook is called we know that the element is rendered and we can get
access to the `nativeElement` of our element:

```typescript
export class DraggableBoxComponent implements AfterViewInit {
    // Inject the ElementRef of this view
    private readonly element = inject(ElementRef);
    
    public ngAfterViewInit(): void {
        // Get access to the native element
        const { nativeElement}  = this.element;
    }
}
```

## Getting access to the document

We will need a reference to the `document` as well since we will listen to events on that element as well.
We will explain later, but we need to inject the `document` inside our component.
Angular provides the `DOCUMENT` injectable for that:

```typescript
export class DraggableBoxComponent implements AfterViewInit {
    // Inject the document
    private readonly document = inject(DOCUMENT);
    ...
}
```

## The event streams

There are three important events that we need if we want to implement drag-and-drop functionality:
- `mousedown`: When we want to start dragging the user has to click the mouse.
- `mousemove`: Every time we move our mouse in drag mode, we want to recalculate the position of the element we are dragging.
- `mouseup`: When we release the mouse after dragging we want to stop the drag-and-drop functionality

There are 2 important DOM nodes that we will use for the drag-and-drop:
- The native element of the drag-and-drop. This will be used to initiate the drag-and-drop functionality. We will listen to the `mousedown` event on that element.
- The `document`. We will use this for the actual dragging and releasing. We will listen to `mousemove` and `mouseup` events on this element. 

It is important to realize why we are listening to the `mousemove` and `mouseup` events of the `document` and not of our native element of the thing we want to drag:
When we would listen to the native element instead of the `document`, and we would move our mouse quickly, chances are big that our cursor would move away from the thing
we were dragging, which would break functionality.

Let's create the actual events now! We can use the `fromEvent` operator from RxJS to create observables from the 
`mousedown`, `mousemove` and `mouseup` events. Since we have to wait until the view is initialized, 
we have to implement these observables inside the `ngAfterViewInit` lifecycle hook:

```typescript
public ngAfterViewInit(): void {
    const nativeElement = this.element.nativeElement;
    const mouseDown$ = fromEvent(nativeElement, 'mousedown');
    const mouseMove$ = fromEvent(this.document, 'mousemove');
    const mouseUp$ = fromEvent(this.document, 'mouseup');
}
```

We see that the `fromEvent` operator takes an element as the first argument and an event it has to listen to as the second argument.

Creating a `dragMove$` observable can be done by leveraging the `switchMap`, `map`, `tap` and `takeUntil` operators:
We start with the `mouseDown$` observable (when the user clicks on the element), We use `switchMap` because it has to start listening
to the `mouseMove$` observable after the user has clicked. We use the `map` operator to calculate the position and the `takeUntil` operator because we want to stop
dragging when the `mouseUp$` observable is triggered.

We can see the basic flow in the code sample below. We haven't implemented the calculation of the position nor the updated position functionality,
but we can see what we are trying to achieve here:

```typescript
public ngAfterViewInit(): void {
    const {nativeElement} = this.element;

    const mouseDown$ = fromEvent(nativeElement, 'mousedown');
    const mouseMove$ = fromEvent(this.document, 'mousemove');
    const mouseUp$ = fromEvent(this.document, 'mouseup');

    const dragMove$ = mouseDown$.pipe(
      switchMap((startEvent: MouseEvent) =>
        mouseMove$.pipe(
          map((moveEvent: MouseEvent) => {
            // Return both events
          }),
          takeUntil(mouseUp$)
        )
      ),
      tap(position => {
        // Update position
      })
    );
    dragMove$.subscribe();
}
```

We notice that we have to subscribe to the `dragMove$` observable in order to make the flow work.
If we don't subscribe, the producer functions will not be called and nothing will happen.

## Calculate the X and Y

To calculate the `x`  and `y` we need the start event (click event) and the move event.
We need to subtract the offset of the start event from the move event. If we do that,
we can calculate the `left` and `top` values and redraw the draggable box.
We will not use Angular to redraw the position because this would not be optimal in terms of performance.
It would mean change detection being triggered on every event, which would result in bad performance.
We rather use vanilla javascript to update the position of the elements.
The `dragMove$` observable now looks like this:

```typescript
public ngAfterViewInit(): void {
    ...
    const dragMove$ = mouseDown$.pipe(
        switchMap((startEvent: MouseEvent) =>
          mouseMove$.pipe(
            map((moveEvent: MouseEvent) => {
              // return both events
              return {
                startEvent,
                moveEvent,
              };
            }),
            takeUntil(mouseUp$)
          )
        ),
        tap(({ startEvent, moveEvent }) => {
          const x = moveEvent.x - startEvent.offsetX;
          const y = moveEvent.y - startEvent.offsetY;
          // update position with vanilla javascript
          nativeElement.style.left = x + 'px';
          nativeElement.style.top = y + 'px';
        })
    );
    dragMove$.subscribe();
}

```

When we look at the example we will see that we will select the text every time we drag which results in bad UX.
To circumvent this we could add a `dragging` class when we drag, that will set the `user-select` property
of CSS to `none`. Every time the mouse is clicked on the element we want to add the `dragging` class, and every time
we release the mouse we want to remove the `dragging` class. For this, we will use Angular since it will only trigger Change
Detection when we start dragging and when we stop dragging.
This means Change Detection will not trigger all the time when we are dragging and only needs to set the class inside a div
in the draggable box.

```typescript
public dragging = false;
...
public ngAfterViewInit(): void {
    ...
    mouseUp$.subscribe(() => {
        this.dragging = false;
    });
    mouseDown$.subscribe(() => {
        this.dragging = true;
    });
}
```

## Avoiding memory leaks

We have a working solution, but we subscribe to 3 observables manually. Since we subscribe manually (not with the async pipe), we need to clean those
subscriptions up manually. If we don't clean those up, we will create memory leaks for those subscriptions.
We will create a local subject of type `void`, that we will next in the `ngOnDestroy` lifecycle hook.
That way we can leverage the `takeUntil` operator which will unsubscribe from the observables for us.

```typescript
export class DraggableBoxComponent implements AfterViewInit, OnDestroy {
    // Create subject of type void
    private readonly destroy$$ = new Subject<void>();
    ...
    public ngAfterViewInit(): void {
        ...
        const mouseDown$ = fromEvent(nativeElement, 'mousedown').pipe(
            takeUntil(this.destroy$$) // clear subscription
        );
        const mouseMove$ = fromEvent(this.document, 'mousemove').pipe(
            takeUntil(this.destroy$$) // clear subscription
        );
        const mouseUp$ = fromEvent(this.document, 'mouseup').pipe(
            takeUntil(this.destroy$$) // clear subscription
        );
        const dragMove$ = mouseDown$.pipe(
            ...
            tap(({ startEvent, moveEvent }) => {
                ...
            }), 
            takeUntil(this.destroy$$) // clear subscription
        );
        ...
    }

    public ngOnDestroy(): void
        // when the components gets destroyed, next the destroy$$
        // subject so the subscriptions get cleaned up
        this.destroy$$.next();
    }
}
```

## Optimizing the Change Detection

By default, all the drag events will trigger Change Detection everywhere.
We see that in the lowest view a drag event is happening that will result in Change Detection being triggered everywhere!

![Change Detection Angular](/assets/angular-performant-drag-and-drop/all-triggered.png)

If we understand how Angular Change Detection works, we should realize that every `mousedown`, `mouseup` but also
`mousemove` event will trigger a `tick()` function on our root view multiple times a second when we are dragging.
This means that every view in our application will get checked for changes all the time.
If you don't know how Change Detection works, you can download this [free Cheat Sheet](https://blog.simplified.courses/angular-change-detection-cheat-sheet-explained/){:target="_blank"}, or learn about it in this [ebook](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}.!
[![Angular Change Detection ebook](/assets/angular-performant-drag-and-drop/ebook.png)](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}.

The first thing we want to do is use the `OnPush` Change Detection Strategy so our
component will not get checked unless it is marked with `LViewFlags.Dirty`:

```typescript
@Component({
    ...
    changeDetection: ChangeDetectionStrategy.OnPush
})
```

This however will not make sure Change Detection isn't triggered on the app level.

We can avoid the Change Detection trigger by running the code that is inside the `ngAfterViewInit()` in the **outer zone** (outside of Angular).
After all, we are updating the position with vanilla js. The only thing that will break is the `dragging` class.
This class will not be switched on and off when we drag because we are not triggering Change Detection anymore.
We can use the `runOutsideAngular()` function of `ngZone` to run code outside of Angular (no Change Detection) and we can use the
`run()` function of `ngZone` to run the code inside the Angular zone (will trigger Change Detection).

We will need to inject `ngZone` for this and optimize our code accordingly:

```typescript
export class DraggableBoxComponent implements AfterViewInit, OnDestroy {
    ...
    // inject NgZone from @angular/core
    private readonly ngZone = inject(NgZone);

    public ngAfterViewInit(): void {
        const {nativeElement} = this.element;
        // fromEvent will cause addEventListener that would normally trigger tick()
        // By running this in the outer zone, tick() will not be called
        // this means change detection will not be called
        this.ngZone.runOutsideAngular(() => {
            const mouseDown$ = fromEvent(nativeElement, 'mousedown').pipe(...);
            const mouseMove$ = fromEvent(this.document, 'mousemove').pipe(...);
            const mouseUp$ = fromEvent(this.document, 'mouseup').pipe(...);
            const dragMove$ = mouseDown$.pipe(...);
            dragMove$.subscribe();
            mouseUp$.subscribe(() => {
                // Here we manually tell angular to run this in ngZone
                // this will trigger tick() => Change Detection
                this.ngZone.run(() => {
                    this.dragging = false;
                });
            });
            mouseDown$.subscribe(() => {
                // Here we manually tell angular to run this in ngZone
                // this will trigger tick() => Change Detection
                this.ngZone.run(() => {
                    this.dragging = true;
                });
            });
        });
    }
    ...
}

```

## Optimizing the Change Detection even more

Now the `tick()` function that triggers application-wide Change Detection will only be called when we start dragging, and when we stop dragging.
For the rest, we will use vanilla javascript to update the dragging of the box, which is more efficient.
However, the `dragging` class is something that is set inside this component, why would we need to trigger Change Detection on the entire application?
We could optimize this by detaching from the ChangeDetectorRef and triggering `detectChanges()` ourselves.

For that, we need to inject the `ChangeDetectorRef` and detach from it. That means that Change Detection will never run for this component.
We can manually trigger Change Detection **for this component only** by running `detectChanges()`.

![Change Detection Angular](/assets/angular-performant-drag-and-drop/local-detectchanges.png){:target="_blank"}.


```typescript
export class DraggableBoxComponent implements AfterViewInit, OnDestroy {
    ...
    private readonly cdRef =  inject(ChangeDetectorRef);

    public ngAfterViewInit(): void {
        const {nativeElement} = this.element;
        this.cdRef.detach(); // we take care of our own Change Detection.
    
        // We still want to avoid this component to trigger application wide 
        // Change Detection so we still want to run it outside of ngZone.
        this.ngZone.runOutsideAngular(() => {
            const mouseDown$ = fromEvent(nativeElement, 'mousedown').pipe(...);
            const mouseMove$ = fromEvent(this.document, 'mousemove').pipe(...);
            const mouseUp$ = fromEvent(this.document, 'mouseup').pipe(...);
            const dragMove$ = mouseDown$.pipe(...);
            dragMove$.subscribe();
            mouseUp$.subscribe(() => {
                this.dragging = false;
                // Run change detection only for this component 
                // (and potential children)
                this.cdRef.detectChanges();
            });
            mouseDown$.subscribe(() => {
                this.dragging = true;
                // Run change detection only for this component 
                // (and potential children)
                this.cdRef.detectChanges();
            });
        });
    }
    ...
}
```

## Conclusion

Now our component is completely independent of Change Detection, and it does not
initiate Change Detection anywhere.

We can see the finished example in this Stackblitz:
[https://stackblitz.com/edit/angular-ivy-heyrwg](https://stackblitz.com/edit/angular-ivy-heyrwg){:target="_blank"}.

Special thanks to the reviewer [Bryan Hannes](https://bryanhannes.com/){:target="_blank"}.
