---
layout: post
title:  "Say goodbye to custom form validators in Angular"
date:   2023-06-02
published: true
comments: true
categories: [Angular, Angular Forms, Angular Signals]
cover: assets/say-goodbye-to-custom-form-validators-in-angular.jpg
description: "This article explains how we can do validations for Angular forms without struggling with boilerplate"
---

**Updated 21 february 2024**

This article is about making form validations:
- straightforward
- clean
- effortless
- opinionated
- **without any boilerplate!!!**

[Check a deep-dive here](https://www.youtube.com/watch?v=vKEd9cNh5R4&t=283s){:target="_blank"}

That's quite the promise right?! Bear with us, we will explain how easy it is and you will never have to write a custom validator ever again. This article is based on [the talk ](https://www.youtube.com/watch?v=EMUAtQlh9Ko){:target="_blank"}
of Ward Bell in combination with our previous article where we explain [template-driven forms or reactive forms](https://blog.simplified.courses/template-driven-or-reactive-forms-in-angular/){:target="_blank"}.
In that article, we explain how we can use Angular to build forms for us automatically. Even though we advise you to use template-driven forms, the approach explained in this article could also be used for Reactive forms!

Whether you love working with reactive forms or template-driven forms... Validations are a hassle for both solutions.

## Template-driven form and reactive form validation issues

With template-driven forms, the problems are:
- Validations are shattered across different templates, there is no single place to find a validation for a form.
- Validations are decorated on the template which results in redundancy and complexity for conditional validators.
- We have to create custom validators.
- We have to create custom directives for those validators.
- Adding and removing validators becomes tricky.

With reactive forms, the problems are:
- The `FormControl` / `FormGroup` composition tends to get dirty.
- Manually removing and adding validators might result in imperative logic.
- We have to build the reactive form manually, while with template-driven forms the creation/removal of `FormControl` and `FormGroup` instances is done for us automatically by the framework.

## What are we validating?

This is an excellent question! Why do we put these validations inside a template? Why would we add it to a form at all?
Does that really make sense, when **we want to validate some kind of model**?
What does validation have to do with Angular? Shouldn't this be a separate process, regardless of which form-solution we use?

We believe it would be better to create a model-validation suite that can be:
- reused in different forms
- reused in the backend
- composable
- easily tested
- comprehended in one single place
- conditional
- framework agnostic

Ward Bell suggests using [Vest](https://vestjs.dev/docs/writing_your_suite/vests_suite){:target="_blank"} suites, that allow us to create functional blocks to validate a specific model. These could be reused through different frameworks, they are composable and have tons of features and even allow conditional validations. If we could create validation suites for all our models, we would gain great flexibility and it would only require a translation vessel to **create Angular validators automatically** based on these suites.

## A model validation suite

Let's say we have a model called `User` that looks like this:

```typescript
export type User = Partial<{
   firstName: string;
   lastName: string;
   passwords: Partial<{
    password: string;
    confirmPassword: string;
   }>

  address: Partial<{
    street: string,
    number: string,
    city: string,
    zipcode: string,
    country: string
  }>;
}>
```

This is a clean model, and the validation suite could look like this:

```typescript
// ./validations/user.validations.ts
import { User } from '../types/user';
import { test, enforce, create } from "vest";

export const userValidations = staticSuite((model: User, field: string) => {
    test('firstName', 'First name is required', () => {
        enforce(model.firstName).isNotBlank();
    });
    test('lastName', 'Last name is required', () => {
        enforce(model.lastName).isNotBlank();
    });
    test(`address.street`, 'Street is required', () => {
        enforce(model.address?.street).isNotBlank();
    });
    test(`passwords.password`, 'Password is required', () => {
        enforce(model.passwords?.password).isNotBlank();
    });
    test(`passwords`, 'Passwords should match', () => {
        enforce(model.passwords.password).equals(
            model.passwords?.confirmPassword
        );
    });
    test(`passwords.password`, 'Should be more than 5 characters', () => {
        enforce(model.passwords?.password).longerThan(5);
    });
});
```

[Check this YouTube video](https://www.youtube.com/watch?v=cuNl-7yu09g&t=67s){:target="_blank"} for a short demo!
*note: the video is not up to date, use staticSuite() instead of create*

The first 2 arguments that the `test()` function takes, is the field and the validation message. It's important here to respect the property structure of our model.

We can see that we write both assertions on properties that would resolve into `FormControl` instances and properties that resolve into `FormGroup` instances.
`firstName` would resolve into a `FormControl` and `passwords` would resolve into a `FormGroup` since it has to validate both the `passwords.password` and `passwords.confirmPassword` properties.

The API of Vest assertions is not in the scope for this article but is explained well in [the docs](https://vestjs.dev/docs/enforce/enforce_rules){:target="_blank"}.

## Conditional validations

Do we need to check if the `passwords` property has more than 5 characters if the password is still empty? No, we don't!
Do we need to compare the two passwords if the `passwords.password` and `passwords.confirmPassword` properties are still blank? No, we don't! For that, Vest has an awesome conditional `omitWhen()` function, that we can use like this:

```typescript
// ./user.validations.ts
...
export const userValidations = staticSuite((model: User, field: string) => {
    ....
    test(`passwords.password`, 'Password is required', () => {
        enforce(model.passwords?.password).isNotBlank();
    });
    // don't check if passwords match if the passwords aren't both filled in
    omitWhen(!model.passwords?.password || !model.passwords?.confirmPassword, () => {
        test(`passwords`, 'Passwords should match', () => {
            enforce(model.passwords?.password).equals(
                model.passwords?.confirmPassword
            );
        });
    });
    // don't check the length if the password isn't filled in yet
    omitWhen(!model.password, () => {
        test(`passwords.password`, 'Should be more than 5 characters', () => {
         enforce(model.passwords?.password).longerThan(5);
        });
    });
});
```

Now, instead of adding, and removing validators, we can have all this logic in a declarative way, in one single place. This is more readable, it's all in one place and it is framework agnostic. What a treat would it be to connect to our favorite framework!

## Composing validations

What about reusability? We might want to reuse the password and address validation somewhere else.
We would love to create some kind of composability like this:

```typescript
import { User } from '../types/user';
import { test, enforce, staticSuite } from 'vest';
import { addressValidations } from './address.validations';
import { passwordValidations } from './password.validations';

export const userValidations = staticSuite((model: User, field: string) => {
  test('firstName', 'First name is required', () => {
    enforce(model.firstName).isNotBlank();
  });
  test('lastName', 'Last name is required', () => {
    enforce(model.lastName).isNotBlank();
  });
  // Reuse an address validation suite that is shared
  addressValidations(model.address, 'address');
  // Reuse a password validation suite that is shared
  passwordValidations(model.passwords, 'passwords');
});
```

We can see that in the `addressValidations()` and `passwordValidations()` functions, we pass an object that is living on our model and as the second argument we pass the `FormGroup` name. Now the implementation of both these validation suites looks like this:

```typescript
// ./address.validations.ts
import { Address } from "../types/address";
import { test, enforce } from "vest";

export function addressValidations(model: Address, field: string): void {
    test(`${field}.street`, 'Street is required', () => {
        enforce(model.street).isNotBlank();
    });
}
```

```typescript
// ./password.validations.ts
import { test, enforce, omitWhen } from "vest";

export function passwordValidations(
    model: {password: string, confirmPassword: string}, field: string
): void{
    test(`${field}.password`, 'Password is required', () => {
        enforce(model.password).isNotBlank();
    });
    omitWhen(!model.password || !model.confirmPassword, () => {
        test(`${field}`, 'Passwords should match', () => {
            enforce(model.password).equals(
                model.confirmPassword
            );
        });
    });
    omitWhen(!model.password, () => {
        test(`${field}.password`, 'Should be more than 5 characters', () => {
            enforce(model.password).longerThan(5);
        });
    });
}
```

## Connecting our vest suites to Angular

### Only validate the field in question

For the first part, we need to use the `only` function to tell vest to only validate the field in question:

```typescript
...
import { ..., only } from 'vest';

export const userValidations = staticSuite((model: User, field: string) => {
    // only execute validation for this field
    only(field);

    test('firstName', 'First name is required', () => { ... });
    ...
});
```

### Making the connection to Vest

I created an entire working solution here where you get access to the entire code:
[Angular Template-driven forms solution](https://blog.simplified.courses/i-opensourced-my-angular-template-driven-forms-solution/){:target="_blank"}

In a nutshell:
- We will use the `ngModel` and `ngModelGroup` selectors
- We will create new directives for them
- Those directives will call parts of or Vest suite.
- I explain the concepts in depth in my [Template-driven forms Course](https://www.simplified.courses/complex-angular-template-driven-forms), but you
  can get the code for free [here](https://blog.simplified.courses/i-opensourced-my-angular-template-driven-forms-solution/){:target="_blank"}

Model validations are easy. They are composable, functional, declarative, easy to read, reusable, testable, framework-agnostic and can be conditional.
This is very nice, but what about that boilerplate reduction?
What does this solution have to do with Angular?
We have to find a way to translate these model validation suites to Angular.

The next thing we want to do is:
- Make sure that not the entire suite is executed on every form change
- Create Angular validator functions automatically based on our model validation suites.
- Add these validator functions automatically to the right `FormControl` and `FormGroup` instances...


## Connecting the dots/Show the errors

We have created our form model suites, we have converted them to Angular validator functions and we found a way to connect these validators to the right `FormControl` and `FormGroup` instances automatically! Now the next step is to show these errors. For that, we are going to create a `scControlWrapper` component that we can use like this:

```html
<form [model]="vm.form" [suite]="suite" ...>
    <label scControlWrapper>
        <span>First name</span>
        <input type="text" [ngModel]="vm.form.firstName" name="firstName" />
    </label>

    <label scControlWrapper>
        <span>Last name</span>
        <input type="text" [ngModel]="vm.form.lastName" name="lastName" />
    </label>
    <app-address ...></app-address>
    <div ngModelGroup="passwords" scControlWrapper>
        <h2>Password</h2>
        <label scControlWrapper>
            <span>Password</span>
            <input
                type="password"
                [ngModel]="vm.form.passwords.password"
                name="password"
            />
        </label>
        <label scControlWrapper>
            <span>Confirm password</span>
            <input
                type="password"
                [ngModel]="vm.form.passwords.confirmPassword"
                name="confirmPassword"
            />
        </label>
    </div>
  </form>
```

We can see that we have **removed all the boilerplate code**! The only 4 things we still need to add to our template are:
- `[ngModel]`
- `ngModelGroup`
- `name`
- `scControlWrapper`

## Wrap up

We learned that both template-driven forms and reactive forms have issues with validators. They are complex and hard to manage.
Model validations make more sense from an architectural point of view and with Vest-suites we can create composable, scalable
and conditional validation suites. By using 2 directives we can easily translate those validation suites to Angular validators and
connect them to the `FormControl` and `FormGroup` instances that are automatically created by Angular.
We created a `scControlWrapper` component that uses the validation errors in combination with content projection to show validation errors
without any boilerplate. As a cherry on top, we refactored the entire form to signals.
I hope you enjoyed this article. Please leave a comment and subscribe for more content.

I want to thank the awesome reviewers of this article:
- [Daniel Glejzner](https://www.twitter.com/danielglejzner){:target="_blank"} 
- [Thomas Laforge](https://www.twitter.com/laforge_toma){:target="_blank"} 
- [Pawel Kubiak](https://www.twitter.com/pawelkubiakdev){:target="_blank"} 

If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training){:target="_blank"} and [Angular Coaching](https://www.simplified.courses/angular-coaching){:target="_blank"}