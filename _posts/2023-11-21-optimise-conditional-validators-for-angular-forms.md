---
layout: post
title: "Optimise conditional validators for Angular Forms"
date: 2023-11-21
published: true
cover: assets/optimise-conditional-validators-for-angular-forms.jpg
comments: true
categories: [Angular, Angular Forms]
description: "In this article, we will optimise the execution of ngValidators for Angular forms"
---

# Intro

In [this video](https://www.youtube.com/watch?v=vKEd9cNh5R4&t=27s){:target="_blank"}, I explained how to write Angular
validations with [Vest.js](https://vestjs.dev/){:target="_blank"} suites.

We have dived into regular validations but also conditional validations.
This example shows that the `confirmPassword` field is only required when the `password` field has a value.
and that both passwords should match but only when they are both filled in.

```typescript
omitWhen(!model.passwords?.password, () => {
    test('passwords.confirmPassword', 'Confirm password is required', () => {
        enforce(model.passwords?.confirmPassword).isNotBlank();
    })
});
omitWhen(!model.passwords?.password || !model.passwords.confirmPassword, () => {
    test('passwords', 'Passwords should match', () => {
        enforce(model.passwords?.password).equals(model.passwords?.confirmPassword);
    });
});
```

Ton connect Angular to Vest, we created a directive that hooks into ngModel.
That directive implements the `Validator` interface and will execute a part of a vest suite when
the `validate()` method is called. [Check the code here](https://github.com/simplifiedcourses/template-driven-forms/blob/main/src/app/template-driven-forms/form-model.directive.ts#L18){:target="_blank"}

Now the issue with Angular is that validations are only run for a control when that control is changed.
So when we type into the `password` field, the system has no idea to run the validator of the `confirmPassword` field.
Since `confirmPassword` should only be validated when `password` changes, it is not executed when the `confirmPassword`
control gets a value.

### We solved this before

We solved this before by using the `alwaysTriggerValidations` input property [that can be found here](https://github.com/simplifiedcourses/template-driven-forms/blob/main/src/app/template-driven-forms/form.directive.ts#L32){:target="_blank"}, but that meant that all the validators
were run every time any control in our form changed. That is not good for performance, and it could be even worse if
we started using asynchronous validations. This is something we will tackle in the next article!

So before we solved the issue like this (using the `alwaysTriggerValidations` input:

```html
<form 
    [alwaysTriggerValidations]="true"
    [formValue]="formValue()" 
    [suite]="suite" 
    (formValueChange)="formValue.set($event)">
</form>
```

### A more efficient solution

What the `alwaysTriggerValidations` input will do, is it will recursively loop over all the controls in our form
and call the `updateValueAndValidity()` method on them. This will result in the entire form to get validated.😅

What we really want, is to only trigger validations when they need to be triggered.
For that reason we can create a `validationConfig` object that tells us exactly which validators need to run
when a certain control is updated.

A part of our validation suite from the previous video looked like this:

```typescript
omitWhen((model.age || 0) >= 18, () => {
    test('emergencyContact', 'Emergency contact is required', () => {
        enforce(model.emergencyContact).isNotBlank();
    })
})
omitWhen(!model.passwords?.password, () => {
    test('passwords.confirmPassword', 'Confirm password is required', () => {
        enforce(model.passwords?.confirmPassword).isNotBlank();
    })
});
```

So when the `age` control changes, we would have to validate the `emergencyContact` control.
When the `passwords.password` control changes, we would have to validate the `passwords.confirmPassword` control.

Our form component would have a `validationConfig` object that look like this:

```typescript
export class SimpleFormComponent {
  protected readonly suite = simpleFormValidations;
  protected readonly formValue = signal<SimpleFormModel>({})
  protected readonly validationConfig: {
    [key: string]: string[]
  } = {
    'age': ['emergencyContact'],
    'passwords.password': ['passwords.confirmPassword']
  }
}
```

And we would update the HTML like this:

```html
<form 
    [validationConfig]="validationConfig"
    [formValue]="formValue()" 
    [suite]="suite" 
    (formValueChange)="formValue.set($event)">
</form>
```

Now the `FormDirective` that hooks into the `form` selector needs get a new input property where we will pass the `validationConfig`.
It would have to loop over all the properties, listen to changes from those properties and update the other
controls when needed.

We can use a setter for that:

```typescript
@Input() public set validationConfig(v: {[key: string]: string[]}){
    // Loop over all the keys
    Object.keys(v).forEach(key => {
        // Listen to changes of the form
        this.ngForm.form.valueChanges.pipe(
            // Only listen to the changes of our key
            map(() => this.ngForm.form.get(key)?.value),
            // Only get notified when there is an actual change
            distinctUntilChanged(),
            // Avoid memory leaks
            takeUntil(this.destroy$$)
        )
        .subscribe((form) => {
            // So the control has changed, let's loop over all the 
            // dependencies of that control
            v[key].forEach((path) => {
                // Trigger the validations of the dependency of that control
                this.ngForm.form.get(path)?.updateValueAndValidity({onlySelf: true, emitEvent: false})
            })
        })
    })
}
```

you can check out [the code here](https://github.com/simplifiedcourses/template-driven-forms/blob/main/src/app/template-driven-forms/form.directive.ts#L40){:target="_blank"}

## Wrapping up

This was a short and simple article, but it had a major impact.
Now only the validators of the controls that actually need to be validated will be run.
We are now ready to introduce Asynchronous validations next monday. It would not make sense
to run asynchronous validations every times a control changes right?! That would result in ajax calls
the entire time.

The entire codebase can be found [here](https://stackblitz.com/edit/stackblitz-starters-sejk7c?file=src%2Fapp%2Fcomponents%2Fsimple-form%2Fsimple-form.component.ts){:target="_blank"}.

Check out the [YouTube video here](https://youtu.be/cvhEPtlttq0){:target="_blank"}