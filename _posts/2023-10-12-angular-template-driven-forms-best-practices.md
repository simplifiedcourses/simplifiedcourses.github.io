---
layout: post
title: "Angular Template-driven Forms Best Practices"
date: 2023-10-12
published: false
comments: true
categories: [Angular, Forms, Best Practices]
description: "This article contains Best Practices on how to use Template-driven Forms in Angular"
---

Template-driven forms are not extremely well documented in Angular, and some people stay away from them because there are no
clear guidelines. In this article I will explain some of the Best Practices I follow when writing big scalable complex forms with Angular Template-driven Forms. These are battle tested in big companies on big projects.

## Ensure a unidirectional dataflow with an empty form value

For this, we need to keep the state of the form in a separate object. You could use a `BehaviorSubject` but to be future-proof
let's put this in an Angular `Signal` called `formValue`. The goal is to never use banana-in-the-box syntax again unless it is for local state.
Since the entire form will be partial anyway, let's start with an empty value `{}` to put in the `formValue` signal.
Here is an example:

```typescript
@Component({ ... })
export class PurchaseFormComponent implements AfterViewInit {
    // Access to the form (that has a reactive form behind the scenes)
    @ViewChild('form') protected ngForm: NgForm | undefined;
    // Create signal that holds the form its value
    // Start empty, angular fill fill it in automatically
    private readonly formValue = signal<PurchaseFormModel>({});

    public ngAfterViewInit(): void {
        this.ngForm!.form.valueChanges.subscribe((v: PurchaseFormModel) => {
            // Update the form value signal if the form changes
            this.formValue.set(v);
        });
    }
    protected submit(): void { ... }
}
```

```html
<h1>Template-driven Purchase form</h1>
<form #form="ngForm" (ngSubmit)="submit()">
  <label>
    <span>First name</span>
    <input type="text" [ngModel]="formValue.firstName" name="firstName" />
  </label>
  <button type="submit">Submit</button>
</form>
<pre>
    {{ formValue | json }}
</pre>
```

### Put the dirty state and/or other pieces of state also in Signals

```typescript

export class PurchaseFormComponent implements AfterViewInit {
    @ViewChild('form') protected ngForm: NgForm | undefined;
    private readonly formValue = signal<PurchaseFormModel>({});
    private readonly formDirty = signal(false);
    private readonly status = signal('VALID');

    public ngAfterViewInit(): void {
        this.ngForm!.form.valueChanges.subscribe((v: PurchaseFormModel) => {
            this.formValue.set(v);
            this.formDirty.set(this.ngForm!.form.dirty);
            this.status.set(this.ngForm!.form.status);
        });
    }
    ...
}
```

## Expose everything through a ViewModel

A ViewModel is a reactive object tailored for the template and also tailored for the template-driven form.
It's a computed signal in our case that we will expose through a getter. Let's start by exposing the 3 states to our template.
We want to expose `formValue`, `status` but we want to disable the reset button if the form is dirty. So for that we will
make our ViewModel a bit more readable and call it `resetDisabled` instead of `formDirty`:

```typescript
export class PurchaseFormComponent implements AfterViewInit {
    ...
    private readonly viewModel = computed(() => ({
        formValue: this.formValue(),
        resetDisabled: !this.formDirty(),
        status: this.status(),
    }));

    protected get vm() {
        return this.viewModel();
    }
}
```

```html
<form #form="ngForm" (ngSubmit)="submit()">
  <label>
    <span>First name</span>
    <input type="text" [ngModel]="vm.formValue.firstName" name="firstName" />
  </label>
  <button type="button" [disabled]="vm.resetDisabled" (click)="reset()">
    Reset
  </button>
  Form is: {{ vm.status }}
  <pre>
        {{ vm.formValue | json }}
    </pre
  >
</form>
```

## Keep the complexity in the class with ViewModels

The biggest selling point for Template-driven forms is: **Let Angular create the entire form structure for you**.
That means your template can contain:

- `ngModel` and `ngModelGroup` directives
- `[disabled]` directives
- `*ngIf` and `*ngFor` statements that Angular can use to build the form for us

That's it. Keep the rest of the form creating and state logic outside your template in your ViewModel.
Here is an example where we do the following:

- We expose the `formValue`, `status` and whether the reset button should be disabled or not.
- The `emergencyContact` field should be disabled when the person is of legal age.
- The `showOtherGender` field should only be in the form when the value of the `gender` field is set to `other`.
- The `shippingAddress` group should only be in the form when the shipping address is different from the billing address.

```typescript
export class PurchaseFormComponent implements AfterViewInit {
    ...
    private readonly viewModel = computed(() => ({
        formValue: this.formValue(),
        status: this.status(),
        resetDisabled: !this.formDirty(),
        emergencyContactDisabled:
            this.formValue().age === 0 || this.formValue().age >= 18,
        showOtherGender: this.formValue().gender === 'other',
        showShippingAddress:
            this.formValue().addresses?.shippingAddressDifferentFromBillingAddress,
    }));
}
```

The HTML could look like this:

```html
<label>
  <span>Emergency contact</span>
  <input
    type="text"
    [ngModel]="vm.formValue.emergencyContactNumber || ''"
    name="emergencyContactNumber"
    [disabled]="vm.emergencyContactDisabled"
  />
</label>
...
<label>
  <span>Specify other gender</span>
  <input
    type="text"
    [ngModel]="vm.formValue.genderOther"
    name="genderOther"
    *ngIf="vm.showOtherGender"
  />
</label>
...
<div ngModelGroup="shippingAddress" *ngIf="vm.showShippingAddress">
  <h2>Shipping address</h2>
  <app-address-form
    [address]="vm.formValue.addresses?.shippingAddress"
  ></app-address-form>
</div>
```

That is pretty readable, pretty declarative and very testable.
All the logic is in the class, the only thing that is in the template tells Angular how to build our Reactive form behind the scenes.

## Don't put any outputs in your templates, listen in the form

Don't listen to `(modelChange)` or other changes in the template. We just want this template to be the single-source-of-truth
for Angular when it comes to building our form. If we want to listen to changes of our form we should do that in the form.
There are multiple approaches

### Synchronous operations

Sometimes we want to set values on the form based on other values of our form.
Let's say that we want to set the `gender` to `male` and the `lastName` to `Billiet` when the `firstName` is `Brecht`...
Let's even say we want to set the `age` to 35, the `gender` to `male`, the `passwords.password` to `Test1234`
and the `passwords.confirmPassword` to `Test12345` when both the `firstName` is equal to `Brecht` and the `lastName` is equal to `Billiet`
This is what it would look like with Reactive forms:

```typescript
this.form.controls.firstName.valueChanges.subscribe((firstName) => {
  if (firstName === "Brecht") {
    this.form.patchValue({ gender: "male", lastName: "Billiet" });
  }
});

combineLatest([
  this.form.controls.firstName.valueChanges,
  this.form.controls.lastName.valueChanges,
]).subscribe(([firstName, lastName]) => {
  if (firstName === "Brecht" && lastName === "Billiet") {
    this.form.patchValue({
      gender: "male",
      age: 35,
      passwords: { password: "Test1234", confirmPassword: "Test12345" },
    });
  }
});
```

Form lookups, custom subscribes, custom patches, and a `combineLatest`.
This isn't very declarative. With Template-driven Forms this is actually very easy to achieve in a declarative way.
Remember that we set the `formValue` signal in the `ngAfterViewInit` lifecycle hook if the value of the form changed?
Check this out!

```typescript
export class PurchaseFormComponent implements AfterViewInit {
    public ngAfterViewInit(): void {
        this.ngForm!.form.valueChanges.subscribe((v: PurchaseFormModel) => {

            // Put declarative logic here...
            if (v.firstName === 'Brecht') {
                v.gender = 'male';
                v.lastName = 'Billiet';
            }
            if (v.firstName === 'Brecht' && v.lastName === 'Billiet') {
                v.age = 35;
                v.gender = 'male';
                v.passwords.password = 'Test1234';
                v.passwords.confirmPassword = 'Test12345';
            }

            this.formValue.set(v);
            ...
        });
    }
}
```

Every time the form is updated, it checks our conditions and updates the form value in a readable way.
Since the value of our form is a new reference anyway, we can just mutate the value here.
If you are a fan of the immutable approach nothing is stopping you to update this value in an immutable way.

### Asynchronous operations

## use ngModel and ngModelGroup correctly, think about your form composition

## Created a typed model

## Use model validations

## Ensure type-safety with schema validations

## Treat them as partial

Angular will create a reactive form behind the scenes for us but not in one go.
It will create it formControl by formControl, formGroup by formGroup.
For that reason it's important to think about your form as a partial object where every property
can be undefined at all times. That makes sense right?! Because in an empty form the values are
not filled in either

## Declarative mutations of the form value

## Split up in child forms

## Disable the entire form or groups with the fieldSet element
