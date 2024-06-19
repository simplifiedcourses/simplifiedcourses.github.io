---
layout: post
title: "I open-sourced my Angular Template-driven Forms Solution"
date: 2023-11-08
published: true
cover: assets/i-opensourced-my-angular-template-driven-forms-solution/banner.jpg
comments: false
authors: [brecht_billiet]
categories: [Angular, Angular Forms, Angular Signals]
description: "In this article, I show you how and why I open-sourced my Angular Template-driven Forms solution"
---

# Intro

If you follow my content, you probably know that I invested a lot of time and energy in Angular Forms
(specifically Template-driven Forms).
I created an [Advanced Template-driven Forms Course](https://www.simplified.courses/complex-angular-template-driven-forms){:target="_blank"}
That uses an opinionated solution that focuses on:
- Boilerplate less code
- Type-safety
- Reactivity
- Declarative code
- Opinionated model validations
- Automatic validation messages translation form Vest to Angular

I care more about making a difference than sales, so I went ahead and open-sourced the entire solution.
**Update! now have created ngx-vest-forms**
Check it out [here](https://www.npmjs.com/package/ngx-vest-forms){:target="_blank"}

I made a nice introduction in [this article](https://blog.simplified.courses/introducing-ngx-vest-forms/){:target="_blank"}

## Demo

That's not all, I created a YouTube video where I show you how to get crazy productive with these forms.
We create a completely validated form with No boilerplate at all:

This is the typescript part of that simple form...

```typescript
@Component({
  selector: 'app-simple-form',
  standalone: true,
  imports: [CommonModule, vestForms],
  templateUrl: './simple-form.component.html',
  styleUrls: ['./simple-form.component.css']
})
export class SimpleFormComponent {
  protected readonly suite = simpleFormValidations;
  protected readonly formValue = signal<SimpleFormModel>({})
}
```

This is the html of my `form` element:

```html
<form scVestForm
    [formValue]="formValue()" 
    [suite]="suite" 
    (formValueChange)="formValue.set($event)">
    ...
</form>
```

And is the amount of code needed to create an input with automatic validation messages:

```html
<div sc-control-wrapper>
    <label>
        <span>First name</span>
        <input type="text" name="firstName" [ngModel]="formValue().firstName">
    </label>
</div>
```

We will create a completely type-safe form that is:
- Unidirectional
- Has a `firstName`, `lastName`, `age`, `emergencyContact` and 2 passwords.
- Has conditional validations
- Has validations on multiple fields (Compare passwords)
- Validates on Blur
- Validates on Submit

This is the result:
![tdd-forms-stackblitz.png](..%2Fassets%2Fi-opensourced-my-angular-template-driven-forms-solution%2Ftdd-forms-stackblitz.png)
I created all that in 5 minutes without **any boilerplate code**!

Check it out here and learn how to become extremely productive with Forms in No-time!
<iframe width="100%" height="500" src="https://www.youtube.com/embed/vKEd9cNh5R4?si=dCA1gLv-KwXPkyQz" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

**Note, this still uses the old solution. We have a stable api now ngx-vest-forms**

Here is the [Stackblitz example](https://stackblitz.com/edit/stackblitz-starters-8zha2s?file=src%2Fapp%2Fcomponents%2Fsimple-form%2Fsimple-form.component.ts){:target="_blank"} from the video.