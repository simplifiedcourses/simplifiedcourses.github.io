---
layout: post
title:  "Model validations for Angular forms without boilerplate"
date:   2023-06-01
published: true
comments: true
categories: [Angular, Forms]
cover: assets/template-driven-or-reactive-forms-in-angular.jpg
description: "This article explains how we can do validations for Angular forms without struggling with boilerplate"
---

This article is about making form validations:
- straightforward
- clean
- effortless
- opinionated
- without boilerplate

That's quite the promise right?! Bear with us, we will explain how easy it is and you will never have to write a custom validator ever again. This article is based on [the talk ](https://www.youtube.com/watch?v=EMUAtQlh9Ko){:target="_blank"}
of Ward Bell in combination with our previous article where we explain [template-driven forms or reactive forms](https://blog.simplified.courses/template-driven-or-reactive-forms-in-angular/){:target="_blank"}.
In that article, we explain how we can use Angular to build forms for us automatically. Even though we advise you to use template-driven forms, the approach explained in this article could also be used for Reactive forms!

Whether you love working with reactive forms or template-driven forms... Validations are always a hassle.

## Template-driven form and reactive form validation issues

With template-driven forms, the problem is:
- Validations are shattered across different templates, there is no single place to find a validation for a form
- Validations are decorated on the template which results in redundancy
- We have to create custom validators
- We have to create custom directives for those validators
- Adding and removing validators becomes tricky

With reactive forms, the problem is:
- The `FormControl` / `FormGroup` composition tends to get dirty
- Manually removing and adding validators might result in imperative logic
- We have to build the reactive form manually, while with template-driven forms the creation/removal of `FormControl` and `FormGroup` instances is done for us automatically.

## What are we validating?

This is an excellent question! Why put these validations inside a template? Why add it to a form at all?
Does that really make sense, when we actually want to validate some kind of model?
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
export type User  = {
   firstName: string;
   lastName: string;
   passwords: {
    password: string;
    confirmPassword: string;
   }

  address: {
    street: string,
    number: string,
    city: string,
    zipcode: string,
    country: string
  };
}
```
This is a clean model, and the validation suite could look like this:

```typescript
import { User } from '../types/user';
import { test, enforce, create } from "vest";

export const userValidations = create((model: User, field: string) => {
    test('firstName', 'First name is required', () => {
        enforce(model.firstName).isNotBlank();
    });
    test('lastName', 'Last name is required', () => {
        enforce(model.lastName).isNotBlank();
    });
    test(`address.street`, 'Street is required', () => {
        enforce(model.address.street).isNotBlank();
    });
    test(`passwords.password`, 'Password is required', () => {
        enforce(model.passwords.password).isNotBlank();
    });
    test(`passwords`, 'Passwords should match', () => {
        enforce(model.passwords.password).equals(
            model.passwords.confirmPassword
        );
    });
    test(`passwords.password`, 'Should be more than 5 characters', () => {
        enforce(model.passwords.password).longerThan(5);
    });
});
```

The first 2 arguments that the `test()` function takes, is the field and the validation message. It's important here to respect the property structure of our model.
We can see that we write both tests on properties that would resolve into `FormControl` instances and properties that resolve into `FormGroup` instances.
`firstName` would resolve into a `FormControl` and `passwords` would resolve into a `FormGroup` since it has to validate both the `password` and `confirmPassword` properties.

The API of Vest is quite readable but is not in the scope of this article.

## Conditional validations

Do we need to check if the `passwords` property has more than 5 characters if the password is still empty? No, we don't!
Do we need to compare the two passwords if the `passwords.password` and `passwords.confirmPassword` properties are still blank? No, we don't! For that, Vest has an awesome conditional `omitWhen` function, that we can use like this:

```typescript
...
export const userValidations = create((model: User, field: string) => {
    ....
    test(`passwords.password`, 'Password is required', () => {
        enforce(model.password).isNotBlank();
    });
    omitWhen(!model.password || !model.confirmPassword, () => {
        test(`passwords`, 'Passwords should match', () => {
            enforce(model.password).equals(
                model.confirmPassword
            );
        });
    }
    );
    omitWhen(!model.password, () => {
        test(`passwords.password`, 'Should be more than 5 characters', () => {
         enforce(model.password).longerThan(5);
        });
    });
});
```

Now, instead of adding, and removing validators, we can have all this logic in a declarative way, in one single place. This is more readable, it's all in one place and it's framework agnostic. What a treat would this be to connect to our favorite framework!

## Composing validations

What about reusability? We might want to reuse the password and address validation somewhere else.
We would love to create some kind of composability like this:

```typescript
import { User } from '../types/user';
import { test, enforce, create } from 'vest';
import { addressValidations } from './address.validations';
import { passwordValidations } from './password.validations';

export const userValidations = create((model: User, field: string) => {
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

We can see that in the `addressValidations()` and `passwordValidations()` function, we pass an object that is living on our model and as the second argument we pass the `FormGroup` name. Now the implementation of both these validation suites looks like this:

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

Model validations are easy. They are composable, functional, declarative, easy to read, reusable, testable, framework-agnostic and can be conditional.
This is very nice, but what about that boilerplate reduction?
What does this solution have to do with Angular? 
We have to find a way to translate these model validation suites to Angular.

The next thing we want to do is:
- Make sure that not the entire suite is executed on every form change
- Create Angular validator functions automatically based on our model validation suites.
- Add these validator functions automatically to the right `FormControl` and `FormGroup` instances...

### Only validate the field in question

For the first part, we need to use the `only` function to tell vest to only validate the field in question:

```typescript
import { User } from '../types/user';
import { test, enforce, create, only } from 'vest';
import { addressValidations } from './address.validations';
import { passwordValidations } from './password.validations';

export const userValidations = create((model: User, field: string) => {
    only(field);

    test('firstName', 'First name is required', () => {
        enforce(model.firstName).isNotBlank();
    });
    ...
});
```

### Create the validator functions automatically

The next thing we need to do is create `ValidatorFn` based on the `field`, the `model` and the `suite`.
Let's create a `createValidator` function for that:

```typescript
import { AbstractControl, ValidatorFn } from "@angular/forms";
import { SuiteResult } from "vest";
import { set } from 'lodash';

export function createValidator<T>(
    field: string,
    model: T,
    suite: (model: T, field: string) => SuiteResult
): ValidatorFn {
    return (control: AbstractControl) => {
        const mod: T = { ...model };

        // this is a neat way to update foo.bar.baz in an object
        // Update the property with path
        set(mod, field, control.value); 
        // Execute the suite with the model and field
        const result = suite(mod, field); 
        // get the errors from our field
        const errors = result.getErrors()[field];
        // expose both an error and errors property
        return errors ? { error: errors[0], errors } : null;
    };
}
```

### Connect it to a FormControl instance

If you are using template-driven forms we can hook into the `[ngModel]` selector and add the validators on that.