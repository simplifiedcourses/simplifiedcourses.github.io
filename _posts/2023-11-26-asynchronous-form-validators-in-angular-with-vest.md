---
layout: post
title: "Asynchronous Form Validators in Angular with Vest"
date: 2023-11-26
published: true
cover: assets/async-form-validators-in-angular-with-vest.jpg
comments: true
categories: [Angular, Angular Forms]
description: "In this article, we will create asynchronous form validators for Angular in our vest suites"
---

# Intro

In this article, we are going to tackle Asynchronous Validations in Angular with Vest.js.

Not a fan of the written word? Check out the YouTube video [here](https://youtu.be/s20RfQJ73tw){:target="_blank"}!

Previously we learned that we can use [vest.js](https://vestjs.dev/){:target="_blank"} to write validation suites.
The advantage of a validation suite is:
- It's framework agnostic
- It's reusable
- It's conditional
- It's composable
- It's testable

Validations become a breeze when using vest.js!
With very limited code we can write beautiful suites that can be reused everywhere in the frontend but also in the backend.

After that, I explained that we use directives to connect angular validator functions to our vest suites. I open-sourced the entire thing and you can play with that [here](https://www.simplified.courses/forms){:target="_blank"}.

We even [optimised these validations suites](https://blog.simplified.courses/optimise-conditional-validators-for-angular-forms/){:target="_blank"} so that we can pass a `validationConfig` object that states which controls their validators should be run when a certain control is updated.

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
export const simpleFormValidations = create((model: SimpleFormModel, field: string) => {
    only(field);
    test('firstName', 'First name is required', () => {
        enforce(model.firstName).isNotBlank();
    });
});
```

We should do:

```typescript
export const createSimpleFormValidations = (swapiService: SwapiService) => {
    return create((model: SimpleFormModel, field: string) => {
        only(field);

        omitWhen(!model.userId, () => {
            test('userId', 'User id is already taken', async ({ signal }) => {
                await lastValueFrom(
                    swapiService
                        .searchUserById(model.userId as string)
                        .pipe(takeUntil(fromEvent(signal, 'abort')))
                ).then(
                    () => Promise.reject(),
                    () => Promise.resolve()
                );
            });
        });
    });
}
```

We could also pass asynchronous functions if we would want to reuse it in another framework:


The component where we call `createSimpleFormValidations()` could look like this:

```typescript
export class SimpleFormComponent {
    protected readonly swapiService = inject(SwapiService);
    protected readonly suite = createSimpleFormValidations(this.swapiService);
    protected readonly formValue = signal<SimpleFormModel>({})
}
```

## Async await

Vest asynchronous validations work with promises. For that we can use the async await syntax.

```typescript
export const createSimpleFormValidations = (swapiService: SwapiService) => {
    return create((model: SimpleFormModel, field: string) => {
        only(field);

        // Only execute when there is a user
        omitWhen(!model.userId, () => {
            // Our test
            test('userId', 'User id is already taken', async () => {
                // Convert to promise
                await lastValueFrom(
                    // Call the backend
                    swapiService.searchUserById(model.userId as string)
                ).then(
                    () => Promise.reject(), // 200 ==> INVALID
                    () => Promise.resolve() // 400 ==> VALID
                );
            });
        });
    });
};
```

When the backend returns a 404, it means the user does not exist, so the field would be valid.
The other way around when the result of the backend is 200, the field should be invalid!

## Abort signal

If we type multiple times this would result in multiple calls that are being made at the same time.
Ideally we want to cancel the previous calls. For that Vest has provided an abort signal for us.
We would use that signal in combination with the `takeUntil` operators from RxJS to stop the call if a new
validation is happening.
We would get the signal as an argument of our asynchronous function:

```typescript
test('userId', 'User id is already taken', async ({ signal }) => {
    ...
});
```
Then on the result observable we would take the `abort` event from our signal and clean it up with `takeUntil`:
This is the final result:

```typescript
omitWhen(!model.userId, () => {
    test('userId', 'User id is already taken', async ({ signal }) => {
        // Convert to promise
        await lastValueFrom(
            swapiService
                .searchUserById(model.userId as string)
                // Clean up open requests
                .pipe(takeUntil(fromEvent(signal, 'abort')))
        ).then(
            () => Promise.reject(), // 200 ==> INVALID
            () => Promise.resolve() // 400 ==> VALID
        );
    });
});
```

## How does it work

To start using this you just need to copy-paste the `templateDrivenForms` directory from [here](https://stackblitz.com/~/github.com/simplifiedcourses/template-driven-forms){:target="_blank"}
We won't go to deep into the internals in this article, we explain it in depth in the course though if the code
wouldn't be self-explanatory. In [this YouTube video](https://youtu.be/s20RfQJ73tw){:target="_blank"}, it is also explained.

In short: We use a custom directive that hooks into the `[ngModel]` selector.
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

You can check the file [here](https://stackblitz.com/edit/stackblitz-starters-uooinv?file=src%2Fapp%2Ftemplate-driven-forms%2Fform-model.directive.ts){:target="_blank"}

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

You can check that [here](https://stackblitz.com/edit/stackblitz-starters-uooinv?file=src%2Fapp%2Ftemplate-driven-forms%2Futils.ts){:target="_blank"}

### Wrapping up

Before I wrap up, have some fun with the [Stackblitz example here](https://stackblitz.com/edit/stackblitz-starters-uooinv){:target="_blank"}

- Asynchronous validators can be great to validate based on data that lives on the server
- We can add asynchronous functions in our vest suites with async await
- There is an abort signal that helps us clean up open ajax calls
- We created async validator logic that is part of [my free boilerplate](https://simplified.courses/forms){:target="_blank"}
- We have no boilerplate anymore and everything just works

I hope you enjoyed, if you do, subscribe, spread, drop me a message 🥰🥰🥰