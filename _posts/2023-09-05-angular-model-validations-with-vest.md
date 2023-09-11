---
layout: post
title:  "Angular form validations with Vest manual"
date:   2023-07-16
published: false
comments: true
cover: assets/state-management-with-angular-signals.jpg
categories: [Angular, Forms, Vest, Angular Signals]
description: "This article explains how to use Angular model validations with Vest"
---

In this article, we will explain how to use Vest model validations in big Angular enterprise solutions.
This is a complete manual on how to use Angular Form validations with Vest.js. We will explain model validations,
and how to connect them to Angular forms without any boilerplate.
[Vest](https://vestjs.dev/){:target="_blank"} is an awesome framework agnostic library that we can use to validate models in an opinionated structured way.

# Why model validations?

Instead of validating Angular forms, React forms, or other forms. We should validate a model instead.
Model validations are framework agnostic, they can be reused in the frontend and backend and they are composable.
Furthermore, these validations are very readable and testable and they follow the separation of concerns and single responsibility principles. In frameworks like Angular, custom validators can be a pain:
- We manually have to add them but also remove them when certain conditionals change
- This can result in imperative and complex code
- Angular validators are tightly coupled to the framework
- Angular validators are shattered across the codebase, there is no one place to maintain them
- Validators don't have access to the entire model of the form


Model validations do have access to the full model, instead of the control that is going to be validated.
This is especially handy when we are creating conditional validations. To understand how to create model validations we recommend reading the [Vest Documentation](https://vestjs.dev/docs/writing_tests/the_test_function){:target="_blank"} but below you can find a simple example of model validation:

```typescript
export const purchaseValidations = create(
  (model: PurchaseFormModel, field: string) => {
    only(field);
    test('firstName', 'First name is required', () => {
      enforce(model.firstName).isNotBlank();
    });
    test('lastName', 'Last name is required', () => {
      enforce(model.lastName).isNotBlank();
    });
});
```

As we can see these are simple assertions, but we can create conditional rules as well:

```typescript
export const purchaseValidations = create(
  (model: PurchaseFormModel, field: string) => {
    ...
    omitWhen(model.age >= 18, () => {
      test(
        'emergencyContactNumber',
        'Emergency contact is required when the age is lower than 18',
        () => {
          enforce(model.emergencyContactNumber).isNotBlank();
        }
      );
    });
    omitWhen(model.gender !== 'other', () => {
      test(
        'genderOther',
        'If gender is other, you have to specify the gender',
        () => {
          enforce(model.genderOther).isNotBlank();
        }
      );
    });
});
```

In the previous example, the `emergencyContactNumber` property will not be validated unless the age is higher than 18, and the `genderOther` property will not be validated unless the gender is set to `other`.
This is especially handy when working with angular form controls that have conditional validators.

Another benefit is these validation rules are composable, so we can make them smaller and more reusable.
Below, we can see that we easily add `phoneNumberValidations` and `addressValidations` to the `purchaseValidations` suite. We will explain how to do this later.

```typescript
export const purchaseValidations = create(
  (model: PurchaseFormModel, field: string) => {
    ...
    // reusable, easy to test separately
    phonenumbersValidations(model?.phonenumbers, 'phonenumbers');
    addressValidations(model.addresses?.billingAddress, 'addresses.billingAddress');
});
```


## Template-driven forms

In this article, we will focus on complex template-driven forms because they contain no boilerplate and we can let Angular do all the work for us. By using a combination of `[ngModel]` `ngModelGroup` and `name` we can create forms in no time and take full advantage of the framework.
If you want to see a comparison between template-driven forms and reactive forms you can [check this article](https://blog.simplified.courses/template-driven-or-reactive-forms-in-angular/){:target="_blank"}.


## The form we will create and validate

For this manual, we have set up a complex purchase form that contains the following functionality:
- Most fields are required.
- When the age is lower than 18, an emergency contact is also required.
- When the gender is set to other, the `genderOther` field is required, otherwise it shouldn't exist.
- The password is always required, but the confirm password is only required when the password is filled in.
- Both the password field and the confirm password field should be the same to make the form valid.
- There should be at least one phone number added.
- Every added phone number should be valid as well.
- The fields of the billing address should be required.
- When the **Shipping address is different from billing address** checkbox is selected, the shipping address should become visible and all its fields should be required as well.
- When the shipping address is exactly the same as the billing address, the form should be invalid as well and show the correct message at the right place
- It should only show the validation errors when a field is touched or when the form is submitted
- It should do all this, without any boilerplate code

Here you can play with the interactive version of the form
[STACKBLITZ DEMO APP HERE]

## Setting up the form

*Note:* To be future-proof, this manual is implemented with Angular Signals, in [this older article](https://blog.simplified.courses/template-driven-or-reactive-forms-in-angular/){:target="_blank"} we explain how to implement it with observables. The principles from this manual can be applied on the older article as well.

Let's start with a very simple version of this form where we add a First name, Last name, Age and Emergency contact.
The emergency contact should be disabled when the age is higher or the same as 18 years old.
Note that we **do not use the banana-in-the-box syntax** for the `ngModel` directive. We want to ensure unidirectional dataflow
and will use the automatically created reactive form to feed the formValue.

This is the html part:

```html
<form
    #form="ngForm"
    (ngSubmit)="submit()"
>
    <label>
        <span>First name</span>
        <input type="text" [ngModel]="vm.formValue.firstName" name="firstName" />
    </label>
    <label>
        <span>Last name</span>
        <input type="text" [ngModel]="vm.formValue.lastName" name="lastName" />
    </label>
    <label>
        <span>Age</span>
        <input type="number" [ngModel]="vm.formValue.age" name="age" />
    </label>
    <label>
        <span>Emergency contact</span>
        <input
            type="text"
            [ngModel]="vm.formValue.emergencyContactNumber"
            name="emergencyContactNumber"
            [disabled]="vm.emergencyContactDisabled"
      />
    </label>
    <div class="buttons">
        <button type="submit">Submit</button>
    </div>
</form>

```

This is the typescript part:

```typescript
export class PurchaseFormComponent implements AfterViewInit {
    // Get access to the form
    @ViewChild('form') ngForm: NgForm | undefined;

    private readonly formValue = signal<PurchaseFormModel>({}); 

    // Calculate our clean and reactive viewModel
    // As a computed signal
    public get vm() {
        return computed(() => {
            formValue: this.formValue(),
            emergencyContactDisabled: this.formValue().age >= 18,
        })();
    }

    public submit(): void { ... }

    public ngAfterViewInit(): void {
        // Listen to the automatically created reactive form
        // and update our signal when the value changes
        // This ensures unidirectional data-flow
        this.ngForm!.form.valueChanges.subscribe((v) => {
            // Set the form value in our signal
            this.formValue.set(v);
        });
    }
}
```

In our complete form, we will have more functionality, more form controls and form groups but let's keep it simple for now.


## Setting up the validations

## Connecting the whole thing

## Wrap up