---
layout: post
title: "Asynchronous Form Validators in Angular with Vest"
date: 2023-11-19
published: true
cover: assets/asynchronous-form-validators-in-angular-with-vest.jpg
comments: true
categories: [Angular, Angular Forms]
description: "In this article, we will create asynchronous form validators for Angular in our vest suites"
---

# Intro

In this article, we are going to tackle Asynchronous Validations in Angular with Vest.js

Previously we learned that we can use [vest.js]() to write validation suites.
The advantage of a validation suite is:
- It's framework agnostic
- It's reusable
- It's conditional
- It's composable
- It's testable

Validations become a breeze when using vest.js!
With very limited code we can write beautiful suites that can be reused everywhere in the frontend but also in the backend.

After that, I explained that we use directives to connect angular validator functions to our vest suites. I open-sourced the entire thing and you can play with that [here]().

We even [optimized these validations suites]() so that we can pass a `validationConfig` object that states which controls their validators should be run when a certain control is updated.

Most of the time this covers all of our use cases but sometimes we need to validate based on data that is living in our backend.
Here are some use cases of when we would need an asynchronous validation:
- Checking if an email address already exists
- If we are working in an appointment-booking form, we might want to check if an appointment slot is still available

## Creating a factory function

An asynchronous validation would perform an asynchronous Ajax call, and based on the result of that call, it would either
validate or invalidate the `formControl` or `formGroup`.
Our Vest suites are completely unrelated to our Angular code, and if we want to make them access some kind of data they would need
access to some services, or at least functions that will trigger Ajax calls behind the scenes.
For that, we can use a factory function.

So instead of doing:

```typescript
import { User } from '../types/user';
import { test, enforce, create } from "vest";

export const userValidations = create((model: FormModel, field: string) => {
    only(field);
    test('firstName', 'First name is required', () => {
        enforce(model.firstName).isNotBlank();
    });
});
```

We should do:

```typescript
export const createUserValidations = (userService: UserService) => {
    return create((model: FormModel, field: string) => {
        only(field);
        test('firstName', 'First name is required', () => {
            enforce(model.firstName).isNotBlank();
        });
    })
};
```

We could also pass asynchronous functions if we would want to reuse it in another framework:

```typescript
export const createUserValidations = (getUser: (userId:string) => Promise<User>) => {
    return create((model: FormModel, field: string) => {
        only(field);
        test('firstName', 'First name is required', () => {
            enforce(model.firstName).isNotBlank();
        });
    })
};
```

The component where we call `createUserValidations()` could look like this:

```typescript
export class Form {
    // Inject the userService
    private readonly userService = inject(UserService);
    private readonly getUser = (userId: string) => lastValueFrom(this.userService.get(userId))
    protected readonly suite = createUserValidations(this.getUser);
}
```

## Async await

Vest asynchronous validations work with promises. For that we can use the async await syntax.

```typescript
export const createUserValidations = (getUser: (userId:string) => Promise<User>) => {
    return create((model: FormModel, field: string) => {
        only(field);
        omitWhen((!model.userId), () => {
            test('userId', 'userId is already taken', async () => {
                await getUser(model.userId as string).then(
                    // User exists, reject because it shouldn't exist
                    () => Promise.reject(),
                    // User does not exist, no validation error
                    () => Promise.resolve()
                );
            });
        });
    })
};
```

## Abort signal

## How does it work

To start using this you just need to copy-paste the `templateDrivenForms` directory from [here]()
We won't go in to depth into the internals in this article, we explain it in depth in the course though if the code
wouldn't be self-explanatory

We use a custom directive that hooks into the `[ngModel]` selector.
That selector implements the `AsyncValidator` interface and gets the following values from the `FormDirective`
that hooks into the `form` selector:
- `ngForm`: A reference to the Angular form
- `suite`: A reference to our validation suite
- `formValue`: The current value of the form (that we need to validate)

We have to implement the `validate()` method. that will get access to the control field and create a new
async validator with the `createAsyncValidator` function. Then we execute the async validator with the control
and we should be good.

```typescript
@Directive({
  selector: '[ngModel]',
  standalone: true,
  providers: [
    // Tell Angular this is an async validator
    { provide: NG_ASYNC_VALIDATORS, useExisting: FormModelDirective, multi: true },
  ],
})
export class FormModelDirective implements AsyncValidator {
    // Inject the FormDirective that holds ngForm, suite and formValue
  private readonly formDirective = inject(FormDirective);

  public validate(control: AbstractControl): Observable<ValidationErrors | null> {
    const { ngForm, suite, formValue } = this.formDirective;
    if (!suite || !formValue) {
      throw new Error('suite or formValue is missing');
    }
    // Get the fieldName eg: addresses.shippingAddress.street
    const field = getFormControlField(ngForm.control, control);
    // Create and execute an async validator
    return createAsyncValidator(field, formValue, suite)(control) as Observable<ValidationErrors|null>;
  }
}
```

You can check the file [here]()

The `createAsyncValidator()` function looks like this:

```typescript
export function createAsyncValidator<T>(
  field: string,
  model: T,
  suite: Suite<string, string, (model: T, field: string) => void>,
): AsyncValidatorFn {
  return (control: AbstractControl) => {
    // Our suite needs the entire model, not just the value
    const mod = cloneDeep(model);
    set(mod as object, field, control.value); // Update the property with path

    return new Observable((observer) => {   
      suite(mod, field).done((result) => {
        // When the validation is finished, get the errors
        // And return next this into the observer
        const errors = result.getErrors()[field];
        observer.next((errors ? { error: errors[0], errors } : null));
        observer.complete();
      })
    })
  };
}
```

You can check that [here]()

### Debouncing the values