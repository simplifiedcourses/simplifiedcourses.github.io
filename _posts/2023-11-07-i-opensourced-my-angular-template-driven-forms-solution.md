---
layout: post
title: "I open-sourced my Angular Template-driven Forms Solution"
date: 2023-11-08
published: true
cover: assets/i-opensourced-my-angular-template-driven-forms-solution/banner.jpg
comments: true
categories: [Angular, Forms, Signals]
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

## 2 versions

### template-driven-forms repo

There is the `template-driven-forms` repo that you can check [here](https://github.com/simplifiedcourses/template-driven-forms){:target="_blank"}
and can test directly [here on Stackblitz codeflow](https://stackblitz.com/~/github.com/simplifiedcourses/template-driven-forms){:target="_blank"}.

The cool thing about this repo is that it contains the entire complex form that I teach about in my [Course](https://www.simplified.courses/complex-angular-template-driven-forms){:target="_blank"}
The demo application has the following functionality:
- Show validation errors on blur
- Show validation errors on submit
- When first name is `Brecht`: Set gender to male
- When first name is `Brecht` and last name is `Billiet`: Set age and passwords
- When first name is Luke: Fetch Luke Skywalker from the swapi api
- When age is below `18`, make Emergency contact required
- When age is of legal age, disable Emergency contact
- There should be at least one phone number
- Phone numbers should not be empty
- When gender is `other`, show Specify gender
- When gender is `other`, make Specify gender required
- Password is required
- Confirm password is only required when password is filled in
- Passwords should match, but only check if both are filled in
- Billing address is required
- Show shipping address only when needed (otherwise remove from DOM)
- If shipping address is different from billing address, make it required
- If shipping address is different from billing address, make sure they are not the same
- When providing shipping address and toggling the checkbox back and forth, make sure the state is kept
- When clicking the Fetch data button, load data, disable the form, and patch and re-enable the form

As you can see, a lot of rules, a lot of conditional stuff, but you can play around with it and you have access to the entire codebase!🥰
This is a screen of the Stackblitz environment:
![tdd-forms-codeflow.png](..%2Fassets%2Fi-opensourced-my-angular-template-driven-forms-solution%2Ftdd-forms-codeflow.png)

### template-driven-forms-starter repo

I also created a second repo: The `template-driven-forms-starter` repo that you can check [here](https://github.com/simplifiedcourses/template-driven-forms-starter){:target="_blank"}
and can test directly [here on Stackblitz codeflow](https://stackblitz.com/~/github.com/simplifiedcourses/template-driven-forms-starter){:target="_blank"}.

This is an empty repo/playground that I created for 2 reasons:
- I can start my YouTube video's starting from this playground ([This is the playlist](https://www.youtube.com/watch?v=djod9on45wc&list=PLTItqHpooUL4SWBZmVIYXCOTFDaOFdU5N){:target="_blank"}).
- It's great to test your knowledge and easily import the template-driven-forms logic directly in your project.

The `src/app/template-driven-forms` directory contains all the code I've created for you to ditch all the boilerplate and to start getting crazy productive with Angular forms!💪

## Demo

That's not all, I created a YouTube video where I show you how to get crazy productive with these forms.
We create a completely validated form with No boilerplate at all:

This is the typescript part of that simple form...

```typescript
@Component({
  selector: 'app-simple-form',
  standalone: true,
  imports: [CommonModule, templateDrivenForms],
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
<form  [alwaysTriggerValidations]="true"
    [formValue]="formValue()" 
    [suite]="suite" 
    (formValueChange)="formValue.set($event)">
    ...
</form>
```

And is the amount of code needed to create an input with automatic validation messages:

```html
<div scControlWrapper>
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

Here is the [Stackblitz example](https://stackblitz.com/edit/stackblitz-starters-8zha2s?file=src%2Fapp%2Fcomponents%2Fsimple-form%2Fsimple-form.component.ts){:target="_blank"} from the video.