---
layout: post
title:  "Running Outputs outside zone.js for Angular performance Optimization"
date:   2022-12-19
published: true
comments: true
categories: Angular, RxJS
cover: assets/running-outputs-outside-zonejs.png

---

In a [previous article](https://blog.simplified.courses/angular-change-detection-onpush-or-not/){:target="_blank"}, we covered the impact of the `OnPush` Change Detection strategy in Angular. We learned that when smartly applying this strategy, we can optimize our performance drastically.

It is also true that this optimization is not always the most important one we can take.
There are 3 ways we can optimize Change Detection in Angular applications:
1. Applying the `OnPush` Change Detection strategy.
2. Detaching a component from the Change Detector with `detach()` and applying our custom strategy by running `detectChanges()` at the right time.
3. Avoiding that Change Detection is running too often.

We will focus on the third optimization in this article, we will avoid too many `tick()` executions by running our code in the outer zone.
`ngZone` has 2 zones: The **inner zone** and the **outer zone**. The inner zone is also called the **Angular zone** and the outer zone is called the **parent zone**. Angular provides us with an injectable called `NgZone` that gives us 2 functions:
- `runOutsideAngular()` that allows us to run code in the outer zone. (Not triggering Change Detection)
- `run()` that allows us to run code back into the inner zone, which will result in Change Detection being executed.

## Event binding triggers Change Detection

When we learn about Change Detection, we realize that binding on an `@Output()` will result in Change Detection being triggered on the root of the application. An output takes an `EventEmitter` or any other kind of observable that can emit events over time.

```html
<my-component (do)="doSomething()"></my-component>
```

```typescript
@Output() do = new EventEmitter();
```

The code snippet above will result in `tick()` functions being executed every time the `do` `@Output()` emits a new event.
`(do)="doSomething()` will create a subscription to the `do` `@Output()` Observable. That subscription will execute an `addEventListener` function which will be captured by zone.js and trigger Change Detection.
For this article, we want to run the previously explained code in the outer zone, so it would not trigger Change Detection.
Basically, we want `@Output()`s that **do not trigger Change Detection** 

## Notifying a parent component in the outer zone

The need for applying the optimizations shown in this article will not occur very often, but in some cases, they might be beneficial to our project.
These optimizations could be needed when our outputs produce a lot of events. Think about a timer or a `mousemove` event. 
`scroll` events are also events that typically emit multiple times a second.
These types of events will trigger Change Detection the moment they emit.

Let's take a `mousemove` event for instance:
`ChildComponent` is responsible for giving its parent the `x` and `y` coordinates of the mouse when the user moves through an `@Output()`.
It listens to the `mousemove` event so every time that event is emitted (which is a dozen times per second), `zone.js` would pick this up and trigger Change Detection globally by executing the `tick()` function.
We could fix that issue by manually subscribing to that event in the outer zone:

```typescript
// Run in the outer zone
ngZone.runOutsideAngular(() => {
    fromEvent(this.document, 'mousemove')
        .subscribe((e: MouseEvent) => {
            this.notifyParent.emit({x: e.x, y: e.y});
        });
});
```

While this would not trigger Change Detection. It makes the use of `@Output()`s ugly and more complex.
It's not reactive and it makes the cleanup for subscriptions harder.
The fact that an `@Output()` can be assigned to any kind of Observable is quite nice.
Take this snippet for example:

```typescript
@Output() public readonly move = fromEvent(this.document, 'mousemove').pipe(
    map((e: MouseEvent) => {
        return {
            x: e.x,
            y: e.y,
        };
    }),
    debounceTime(1000),
    filter(...)
);
```

We can see that we can create neat reactive flows, that are easy to read.
There is one downside here: We will constantly be triggering Change Detection on the entire application when the user moves his mouse in this component.
Somehow we want to make sure this code runs in the **outer zone** of Angular so that this does not result in useless Change Detection cycles.

## Our example

To clarify this more, we have created an example.

We have a child component called `ChildComponent` that is responsible for notifying the parent component called `ParentComponent` with the `x` and `y` coordinates as the mouse of the user moves.
At the same time, `ParentComponent` keeps a property called `val` that is incremented by `ChildComponent` every second.

Even though `val` changes every second, and the parent is constantly getting notified with the latest value of `x`  and `y`, we only want to update the DOM when the mouse of the user is moving inside the green square. The green box is located in the upper left corner so when the `x` value and the `y` value are 100 or lower, Change Detection will get triggered on the `ParentComponent` (**not on the root of the application**)

This is the code with the default Angular Change Detection behavior. It does exactly what we want but it will Change Detection way too often!

```typescript
// ChildComponent
@Component({
    selector: 'child',
    standalone: true,
    changeDetection: ChangeDetectionStrategy.OnPush,
    template: `
        <h1>{{val}}</h1>
        <div class="square"></div>
    `,
    ],
})
export class ChildComponent {
    private document = inject(DOCUMENT);
    @Input() public val: number;

    // Example of simple observable bound to an @Output()
    @Output() public readonly valChange = interval(1000);

    // Example of observable with extra operator bound to an @Output()
    @Output() public readonly move = fromEvent(this.document, 'mousemove')
        .pipe(
            map((e: MouseEvent) => {
                return {
                    x: e.x,
                    y: e.y,
                };
            })
        );
}

// ParentComponent
@Component({
    selector: 'parent',
    standalone: true,
    imports: [ChildComponent, CommonModule],
    changeDetection: ChangeDetectionStrategy.OnPush,
    template: `
        <child
            [val]="val"
            (move)="onMove($event)"
            (valChange)="onValChange($event)">
        </child>
        <strong>X: {{mouseX}}, Y:{{mouseY}}</strong>
  `,
})
export class ParentComponent {
    public val = 0;
    public mouseX = null;
    public mouseY = null;

    public onValChange(val: number): void {
        this.val = val;
    }

    public onMove(mouseCoordinates: { x: number; y: number }): void {
        this.mouseX = mouseCoordinates.x;
        this.mouseY = mouseCoordinates.y;
    }
}
```

[Check the Stackblitz example](https://stackblitz.com/edit/angular-ivy-rq6r7k?file=src%2Fapp%2Fchild.component.ts){:target="_blank"}. We see that wherever we move our mouse on the screen the values are being updated. We only want to update the values/trigger Change Detection when we are moving the mouse in the green square.

### Quick recap

At this moment we are triggering Change Detection on the root of our application **a dozen times per second**. If we would have a big application that has a lot of components this could be problematic and even cause the application to crash.

## Binding Angular @Output() in the outer zone

We don't want to manually subscribe and run things in the outer zone... We want to keep the **reactive flow**. For that reason, we are going to create a custom operator that runs the subscription of this `@Output()` in the outer zone, so that Change Detection will not be triggered anymore.
We will create an `runOutsideAngular()` operator that we can use like this:

```typescript
@Output() public readonly valChange = interval(1000)
    // Subscribes to the source observable in the outer zone
    .pipe(runOutsideAngular());

@Output() public readonly move = fromEvent(this.document, 'mousemove')
    .pipe(
        runOutsideAngular(), // Subscribes to the source observable
                          // in the outer zone
        map((e: MouseEvent) => {
            return {
                x: e.x,
                y: e.y,
            };
        })
    );
```

Creating a custom operator in RxJS is quite straightforward. It's a function
that returns a function that takes the source observable as a parameter.

```typescript
export const outsideAngular = <T>() => (source$: Observable<T>) => {
    // Todo: create yourNewObservable$
    return yourNewObservable$;
};
```

In our case, we want to subscribe on `source$` inside the **outer zone** of Angular and keep the results of that subscription in a subject that we can return. 
For that, we need to get access to an instance of `NgZone`... We can use the `inject()` function of Angular to get access to that instance.

```typescript
export const runOutsideAngular = <T>() => (source$: Observable<T>) => {
    // Get access to ngZone
    const ngZone = inject(NgZone);

    // Create a subject of the same generic type
    // as the source observable
    const sub$$ = new Subject<T>();
    
    // Declare the subscription here, we need to clean it up later
    let subscription: Subscription;

    // Run the actual subscription that executes an
    // addEventListener outside of Angular, so that
    // Change Detection won't be triggered automatically
    ngZone.runOutsideAngular(() => {
        subscription = source$.subscribe(sub$$);
    });

    // Return the subject but be sure to unsubscribe from
    // the source observable when the subject completes
    // or errors
    return sub$$.pipe(
        finalize(() => {
            subscription.unsubscribe();
        })
    );
};
```

Everything works as expected now: `ChildComponent` does not trigger Change Detection anymore but `ParentComponent` is notified with the new values.

We can see that in this [StackBlitz example](https://stackblitz.com/edit/angular-ivy-vecsk5?file=src%2Fapp%2Fchild.component.ts){:target="_blank"}
The DOM is not rerendered, but `console.log` statements are being made every second and when the user moves the mouse.

## Custom Change Detection

`ChildComponent` does what it needs to do:
- Run event listeners in the outer zone
- Notify the parent through `@Output()`s without triggering Change Detection

`ParentComponent` is now responsible for triggering Change Detection and we don't want to trigger it all the time. We only want to trigger it when the mouse for the user moves within the green box.


```typescript
export class ParentComponent {
    // Inject ChangeDetectorRef because we want to
    // manually trigger Change Detection with the
    // detectChanges() function
    private readonly changeDetectorRef= inject(ChangeDetectorRef)
    public val = 0;
    public mouseX = null;
    public mouseY = null;

    public onValChange(v: number): void {
        // Set the value of val, but don't trigger
        // Change Detection
        this.val = v;
        console.log(v);
    }

    public onMove(mouseCoordinates: { x: number; y: number }): void {
        // Set the values always
        this.mouseX = mouseCoordinates.x;
        this.mouseY = mouseCoordinates.y;
        // Only trigger Change Detection when the event
        // occured inside the green box
        if(mouseCoordinates.x <= 100 && mouseCoordinates.y <= 100){
            this.changeDetectorRef.detectChanges();
        }
        console.log(mouseCoordinates);
    }
}
```

We can find the optimized Stackblitz example [here](https://stackblitz.com/edit/angular-ivy-vbejhq?file=src%2Fapp%2Fchild.component.ts){:target="_blank"}

## Conclusion

We learned that there are 3 ways of optimizing for Change Detection.
1. The first way is by [using the OnPush Change Detection strategy which is described in detail in this article](https://blog.simplified.courses/angular-change-detection-onpush-or-not/){:target="_blank"}. 
2. Detaching a component completely from the Change Detector 
3. Run code in an outer zone
   
We saw how we could achieve this by running `ngZone` its `runOutsideAngular()` function but that was hard to combine with reactive `@Output()`s.

Using Observables on `@Output()`s  gives us the ability to create reactive flows and by creating a `runOutsideAngular()` operator we were able to create `@Output()`s that notify their parents but do not initiate Change Detection.

While that is probably not something we want to use for everything, it might be beneficial for events that trigger Change Detection too much.

If you like the article, leave me a comment below and if you want to learn more about Change Detection, you might be interested in buying this [Angular Change Detection book](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}. It will help you understand and resolve performance issues in your Angular Enterprise projects.

Special thanks to the reviewers:
- [Bryan Hannes](https://bryanhannes.com){:target="_blank"}
- [Gregor Woiwode](https://twitter.com/gregonnet){:target="_blank"}