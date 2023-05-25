---
layout: post
title:  "CHATGPT helped me write a snake game in angular and how I optimized it for performance"
date:   2023-01-08
published: true
comments: true
categories: [Angular, ChatGPT, Change Detection]
cover: assets/chatgpt-snake-game.jpg
description: "This article shows we can use CHATGPT to help use code and in this case write a snake game in Angular."

---

## What is CHATGPT?

In November 2022, [OpenAi](https://openai.com/){:target="_blank"} launched CHATGPT and made it public for everyone. CHATGPT comes with APIs and a chat interface that we can use to talk to the AI.
We can ask it anything:
- How to teach or dog to do a head roll
- Any type of mathematical equations
- How Angular Change Detection works behind the scenes
- How to negotiate a new pay raise
- To write a blog article for us (beware this could hurt us in terms of SEO since Google uses AI tracking tools and will punish your website if your content is written by an AI)
- A joke, a fun fact, a Typescript tip
- etc...
  
We can also ask it to code for us:
<blockquote>Write me a Fibonacci program in javascript</blockquote>
CHATGPT will write us a Fibonacci program with code samples and explanations. Cool!
<blockquote>Now write me without using for loops</blockquote>
![CHATGPT Fibonacci](/assets/how-chatgpt-helped-me-write-a-snake-game-in-angular-and-how-i-optimized-it-for-performance/fibonacci.png)

This can be super helpful and today we will talk about how CHATGPT can help us to create a **snake game** and how we can optimize it for performance.

## Asking CHATGPT to write the snake game

A snake game can be very specific, can be quite limited and there are tons of variations. Also, it is worth noting that every answer that CHATGPT gives us can differ from the previous one. So let's start by asking: **"Write me the most complete snake game in Angular"**

**CHATGPT replies** : Sure! Here is a complete Snake game implemented in Angular:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <canvas #canvas width="400" height="400" style="border: 1px solid black;"></canvas>
  `,
  styles: []
})
export class SnakeGameComponent {
  @ViewChild('canvas') canvas: ElementRef<HTMLCanvasElement>;
  private ctx: CanvasRenderingContext2D;
  private snake: {x: number, y: number}[] = [];
  private direction: 'up' | 'down' | 'left' | 'right' = 'right';
  private food: {x: number, y: number};
  private gameOver = false;
  private score = 0;

  ngAfterViewInit() {
    this.ctx = this.canvas.nativeElement.getContext('2d');
    this.resetGame();
    setInterval(() => this.tick(), 100);
  }

  private resetGame() {
    this.snake = [{x: 0, y: 0}];
    this.direction = 'right';
    this.food = {
      x: Math.floor(Math.random() * this.canvas.nativeElement.width / 10) * 10,
      y: Math.floor(Math.random() * this.canvas.nativeElement.height / 10) * 10
    };
    this.gameOver = false;
    this.score = 0;
  }

  private tick() {
    if (this.gameOver) {
      return;
    }
    this.advanceSnake();
    this.draw();
  }

  private advanceSnake() {
    const head = this.snake[0];
    let newHead: {x: number, y: number};
    if (this.direction === 'up') {
      newHead = {x: head.x, y: head.y - 10};
    } else if (this.direction === 'down') {
      newHead = {x: head.x, y: head.y + 10};
    } else if (this.direction === 'left') {
      newHead = {x: head.x - 10, y: head.y};
    } else if (this.direction === 'right') {
      newHead = {x: head.x + 10, y: head.y};
    }
    if (this.outOfBounds(newHead) || this.snakeCollision(newHead)) {
      this.gameOver = true;
      return;
    }
    this.snake.unshift(newHead);
    if (this.foodCollision(newHead)) {
      this.score++;
      this.spawnFood();
    } else {
      this.snake.pop();
    }
  }

  private outOfBounds(position: {x: number, y: number}) {
    return (
      position.x < 0 ||
      position.y < 0 ||
      position.x >= this.canvas.nativeElement.width ||
      position.y >= this.canvas.nativeElement.height
    );
  }

  private snakeColl
  ```

  CHATGPT can only emit a certain number of characters so we see that the program abruptly ends with `private snakeColl`.

  This is because the result is limited. We can just type `continue` and CHATGPT will continue writing the program for us.

  **CHATGPT replies:** Sure! Here is the rest of the Snake game code:

  ```typescript
   private snakeCollision(position: {x: number, y: number}) {
    return this.snake.some(segment => segment.x === position.x && segment.y === position.y);
  }

  private foodCollision(position: {x: number, y: number}) {
    return this.food.x === position.x && this.food.y === position.y;
  }

  private spawnFood() {
    this.food = {
      x: Math.floor(Math.random() * this.canvas.nativeElement.width / 10) * 10,
      y: Math.floor(Math.random() * this.canvas.nativeElement.height / 10) * 10
    };
  }

  private draw() {
    this.ctx.clearRect(0, 0, this.canvas.nativeElement.width, this.canvas.nativeElement.height);
    this.ctx.fillStyle = 'black';
    this.snake.forEach(segment => this.ctx.fillRect(segment.x, segment.y, 10, 10));
    this.ctx.fillStyle = 'red';
    this.ctx.fillRect(this.food.x, this.food.y, 10, 10);
    this.ctx.font = '20px Arial';
    this.ctx.fillStyle = 'black';
    this.ctx.fillText(`Score: ${this.score}`, 10, 25);
    if (this.gameOver) {
      this.ctx.fillText('Game Over', 150, this.canvas.nativeElement.height / 2);
    }
  }

  @HostListener('window:keydown', ['$event'])
  keyEvent(event: KeyboardEvent) {
    if (event.key === 'ArrowUp' && this.direction !== 'down') {
      this.direction = 'up';
    } else if (event.key === 'ArrowDown' && this.direction !== 'up') {
      this.direction = 'down';
    } else if (event.key === 'ArrowLeft' && this.direction !== 'right') {
      this.direction = 'left';
    } else if (event.key === 'ArrowRight' && this.direction !== 'left') {
      this.direction = 'right';
    }
  }
}
```

Now it gives some extra feedback telling us that the snake game uses a canvas element and uses a canvas 2D rendering context to draw the snake and food.
It also gives us some extra intel.

So we have our initial program. Let's create a Stackblitz example to see if this code actually works.
When copy-pasting the generated code into the Stackblitz example the compiler gives us three errors:

The program is not working, there are some things missing:
- Cannot find name 'ViewChild'
- Cannot find name 'ElementRef'
- Cannot find name 'HostListener'

We already know that our application is missing imports on the top of our application but let's make CHATGPT work for us:
Let's ask it the following question:
![CHATGPT Angular missing imports](/assets/how-chatgpt-helped-me-write-a-snake-game-in-angular-and-how-i-optimized-it-for-performance/missing-import.png)

CHATGPT gives us some very valuable feedback, but we are frustrated now and we want to know why it didn't add that in the first place.
Let's ask, shall we?
![CHATGPT Angular apologize](/assets/how-chatgpt-helped-me-write-a-snake-game-in-angular-and-how-i-optimized-it-for-performance/apologise.png)

This is truly amazing! It not just apologizes but actually understands that there might also be other imports missing. Let's add the missing imports to our Stackblitz example and see if we can make the game work.
This [Stackblitz example](https://stackblitz.com/edit/angular-ivy-7cchxc?file=src%2Fapp%2Fapp.component.ts,src%2Fapp%2Fapp.module.ts,src%2Findex.html){:target="_blank"} already works and is completely written by CHATGPT.


## Letting CHATGPT optimize the code for us.

This terrific AI has created a fully working snake game for us. We just had to copy-paste it into Stackblitz.
Let's take it one step further. We don't feel comfortable with the fact that CHATGPT has generated all of the code into one file.
We have been taught that separation of concerns is a good thing and we want to know more about that.

![CHATGPT Angular service](/assets/how-chatgpt-helped-me-write-a-snake-game-in-angular-and-how-i-optimized-it-for-performance/service.png)

After typing `continue` a few more times, CHATGPT has now spit out a complete snake game that:
- has all the imports we need (it remembered)
- split up the code into a component and service
- It got better, it implemented the `AfterViewInit` interface and in the previous version, it didn't.

CHATGPT isn't flawless. it forgot to add the `getDirection()` function and it was missing the `canvasWidth` and `canvasHeight` properties.
After telling CHATGPT that `getDirection()` was missing it give me an updated example.
We quickly fixed the `canvasWidth` and `canvasHeight` properties as well and we now have a fully working Stackblitz example [here](https://stackblitz.com/edit/angular-ivy-b2xad2?file=src%2Fapp%2Fapp.component.ts,src%2Fapp%2Fapp.module.ts,src%2Findex.html,src%2Fapp%2Fsnake-game.service.ts,src%2Fapp%2Fapp.component.css){:target="_blank"}.

## Taking it to the next level

CHATGPT created a game for us, and optimized it for us, it isn't perfect but it helps us fix its mistakes. Let's ask CHATGPT how we can optimize the game for performance. This is a small summary of what it suggests:
- Using requestAnimationFrame instead of setInterval
- Only redrawing the portion of the canvas that has changed
- Use object pooling
- Use web workers

That's not really what we were looking for although they seem like solid tips. 

I was nosing around before in the performance tab and I saw that `detectChanges()` was called every **100ms**, and took **12ms** every time as we can see in this image: 

![CHATGPT Angular change detection called](/assets/how-chatgpt-helped-me-write-a-snake-game-in-angular-and-how-i-optimized-it-for-performance/cdran.png){:target="_blank"}

The reason is that the `setInterval()`  will trigger Angular Change detection every **100ms** because of zone.js.
```typescript
ngAfterViewInit() {
    this.ctx = this.canvas.nativeElement.getContext('2d');
    this.snakeGame.resetGame();
    setInterval(() => this.tick(), 100);
}
```

Let's dig deeper and ask CHATGPT: **Something Angular Change Detection related maybe?** 


Now CHATGPT tells us to use the [OnPush strategy](https://blog.simplified.courses/angular-change-detection-onpush-or-not/){:target="_blank"}
That is a solid tip, we don't want to trigger Change Detection on the app component every **100ms**. We are using a canvas anyway so we don't need Change Detection there, so adding the `ChangeDetectionStrategy.OnPush` will result in better performance

```typescript
@Component({
  ...
  changeDetection: ChangeDetectionStrategy.OnPush
})
```
When we measure the performance of the updated [Stackblitz example](https://stackblitz.com/edit/angular-ivy-kx12c2?file=src%2Fapp%2Fapp.component.ts,src%2Fapp%2Fapp.module.ts,src%2Findex.html,src%2Fapp%2Fsnake-game.service.ts,src%2Fapp%2Fapp.component.css){:target="_blank"} we can see that Change Detection takes less time: This time it only takes **18μs** (**microseconds, not milliseconds**)
![CHATGPT Angular Change detection OnPush](/assets/how-chatgpt-helped-me-write-a-snake-game-in-angular-and-how-i-optimized-it-for-performance/cdranlean.png)

We can see that `detectChanges()` is still being called every 100ms, but it became super cheap since there is not a single component that needed to be detected for changes.

Check the [Stackblitz example](https://stackblitz.com/edit/angular-ivy-kx12c2?file=src%2Fapp%2Fapp.component.ts){:target="_blank"} to see the updated version.

## The human part

What CHATGPT didn't know yet is that Change Detection actually never should be run...
We could run the entire game in the **Angular outer zone**.
This is where the reasoning of real developers comes in, instead of machines. Let's run the game outside of Angular (explained in [this article](https://blog.simplified.courses/angular-performant-drag-and-drop-with-rxjs/#optimizing-the-change-detection){:target="_blank"}):

```typescript
export class SnakeGameComponent implements AfterViewInit {
  private ngZone = inject(NgZone);
  ...

  ngAfterViewInit() {
    // run outside of angular so Change Detection is
    // never executed
    this.ngZone.runOutsideAngular(() => {
      this.ctx = this.canvas.nativeElement.getContext('2d');
      this.snakeGame.resetGame();
      setInterval(() => this.tick(), 100);
    })
  }
  ...
```

This results into Change Detection being completely removed as we can see in this example:
![CHATGPT Angular snake optimized](/assets/how-chatgpt-helped-me-write-a-snake-game-in-angular-and-how-i-optimized-it-for-performance/optimized.png)
[Here](https://stackblitz.com/edit/angular-ivy-dmn3uj?file=src%2Fapp%2Fapp.component.ts){:target="_blank"} is the final and optimized Stackblitz example

## Conclusion

CHATGPT is awesome and can be used to:
- Copy write
- Ask questions to
- Make small essays
- Write games for us
- Help us debug where there is something wrong
- help us optimize or code

It's still not as perfect as a real developer, that has the experience and knows how to reason about things.
That being said, it is a great colleague and can help us out way better than a search engine would at some times.

By optimizing Change Detection we can optimize the performance of our application greatly.
In this case, we didn't even need Change Detection, so we were able to run the game in the outer zone.
If you like to learn about Angular Change Detection, we highly recommend you to read our [book about Angular Change Detection](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}. We put a lot of love and effort into it and it is backed up with Stackblitz examples.
If you liked the article, please leave a comment!

If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training) and [Angular Coaching](https://www.simplified.courses/angular-coaching)