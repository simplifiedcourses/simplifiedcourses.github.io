---
layout: post
title: "Angular Template-driven Forms state management"
date: 2023-10-26
published: true
cover: assets/template-driven-forms-state-management.jpg
comments: true
categories: [Angular, Angular Forms, Angular Signals, State management]
description: "In this article, we will tackle how to handle state management when using Template-driven Forms in Angular"
---

# Intro
**Updated 21 february 2024**


In this article, we will tackle how to handle state management when using Template-driven Forms in Angular.
A big difference in terms of state management between Template-driven forms and Reactive Forms in Angular is that with Template-driven Forms...
**We let Angular do all the work for us!! (including state management)**
In practice, that means:

- Angular will create form control and form group instances for us automatically.
- Angular will remove form control and form group instances for us automatically.
- We never have to manually add/remove validators.
- Our form will only contain the controls and groups that it needs.

[Interested in the YouTube video? Check it here](https://youtu.be/djod9on45wc){:target="_blank"}

## The problem with automatic state management

Think about the next example:

We have a form that has a form group called `addresses` that has a `billingAddress` and an optional `shippingAddress` form groups.
The user has the ability to optionally provide the `shippingAddress` when a
`shippingAddressIsDifferentThanBillingAddress` property that is bound to a checkbox is set to true.
This means that when the checkbox is not selected, the `shippingAddress` should not be part of the form
and its validators should not be executed either.
Read [This article or YouTube videos](https://blog.simplified.courses/angular-template-driven-forms-with-signals/){:target="_blank"} if you don't understand how we created the `form` directive and how the ViewModel works:


```typescript
export type PurchaseFormModel = Partial<{
  firstName: string;
  lastName: string;
  gender: 'female'| 'male'| 'other';
  genderOther: string;
  addresses: Partial<{
    billingAddress: AddressFormModel;
    // should not always be shown
    shippingAddress: AddressFormModel;
    shippingAddressDifferentFromBillingAddress: boolean;
  }>
}>;


export class MyFormComponent {
  // Hold the value of the unidirectional form
  private readonly formValue = signal<PurchaseFormModel>({});

  // Declarative ViewModel that exposes the form value
  // and whether the shipping address should be shown
  protected readonly viewModel = computed(() => ({
    formValue: this.formValue(),
    // Declarative property on when to show the shipping address or not
    showShippingAddress:
      this.formValue().addresses?.shippingAddressDifferentFromBillingAddress,
  }));

  // Expose vm as getter for shorter syntax
  protected get vm() {
    return this.viewModel();
  }

  protected setFormValue(e: PurchaseFormModel): void {
    // Ensure unidirectional dataflow
    this.formValue.set(e);
  }
}
```
The HTML of this code could look like this :

```html
<form (formValueChange)="setFormValue($event)">
  ...
  <div ngModelGroup="addresses">
    <h3>Billing address</h3>
    <app-address
      ngModelGroup="billingAddress"
      [address]="vm.formValue.addresses?.billingAddress"
    >
    </app-address>
    <input
      type="checkbox"
      [ngModel]="vm.formValue.addresses?.shippingAddressDifferentFromBillingAddress"
      name="shippingAddressDifferentFromBillingAddress"
    />
    <div *ngIf="vm.showShippingAddress">
      <h3>Shipping address</h3>
      <app-address
        ngModelGroup="shippingAddress"
        [address]="vm.formValue?.addresses?.shippingAddress"
      >
      </app-address>
    </div>
  </div>
</form>
```

Angular will turn this HTML automatically in the following dynamic form controls and form groups behind the scenes:

```typescript
form = {
    addresses: FormGroup = {
        billingAddress: FormGroup = {
            street: FormControl,
            number: FormControl,
            country: FormControl,
            city: FormControl,
            zipcode: FormControl
        }
        shippingAddressDifferentFromBillingAddress: FormControl
    }
}
```

The advantage of using Template-driven forms is that Angular takes care of everything automatically for us.
The state is lost because it's kept in the Template-driven Form that no longer has the `shippingAddress` form group. 
It was automatically destroyed together with the form control instances of the `shippingAddress`.
The problem here is that when you select the checkbox, provide the `shippingAddress` and toggle the checkbox to `false` and `true` again, that the state is lost. 
Even though that is what you want in most cases, there are use-cases where we want to keep that state.
When the `shippingAddressDifferentFromBillingAddress` property is set to `false` the value of the form looks like:

```json
{
  "addresses": {
    "billingAddress": {
      "street": null,
      "number": null,
      "city": null,
      "zipcode": null,
      "country": null
    },
    "shippingAddressDifferentFromBillingAddress": false
  }
}
```

When the `shippingAddressDifferentFromBillingAddress` property is set to `true` the value of the form looks like:

```json
{
  "addresses": {
    "billingAddress": {
      "street": null,
      "number": null,
      "city": null,
      "zipcode": null,
      "country": null
    },
    "shippingAddressDifferentFromBillingAddress": true,
    "shippingAddress": {
      "street": null,
      "number": null,
      "city": null,
      "zipcode": null,
      "country": null
    }
  }
}
```

## Keeping state outside of the form

We can see that `shippingAddress` is automatically added and removed.
When that could be seen as expected behavior, in some cases we do want to keep the state.
Since that state does not have anything to do with the validation state of the form we can extract it from that form entirely and put that in a signal.

```typescript
export class MyFormComponent {
  // Keep the shippingAddress in a local state signal
  private readonly shippingAddress = signal<AddressFormModel({});
  ...

  protected readonly viewModel = computed(() => ({
    ...
    // Calculate a shippingAddress property that takes the shippingAddress
    // from the formValue, and when it does not exist it should take it from
    // the shippingAddress signal as a fallback, ensuring the persistance of the state
    shippingAddress: this.formValue().addresses?.shippingAddress || this.shippingAddress()
  }));

  ...

  protected setFormValue(e: PurchaseFormModel): void {
    this.formValue.set(e);
    // if there is a shippingAddress, update the local state
    if (e.addresses?.shippingAddress) {
      this.shippingAddress.set(e.addresses.shippingAddress);
    }
  }
}
```

We just added a new signal containing the state, added a computed property called `shippingAddress`
and set the state if there was an update with the `shippingAddress`.
Now let's update the HTML to bind to `shippingAddress` instead of the `shippingAddress` living in `formValue.addresses.shippingAddress`:

```html
<form (formValueChange)="setFormValue($event)">
  ...
  <div ngModelGroup="addresses">
    ...
    <div *ngIf="vm.showShippingAddress">
      <h3>Shipping address</h3>
      <!-- use vm.shippingAddress directly -->
      <app-address ... [address]="vm.shippingAddress"> </app-address>
    </div>
  </div>
</form>
```

Now Angular do the following things for us:

- When initially the checkbox is set to `false`:
  - There will be no `shippingAddress` form group.
  - There will be no form group for the `shippingAddress`.
- When the checkbox is set to `true`:
  - The `shippingAddress` form group and its form controls will be added.
  - All their potential validators will be executed.
- When the user starts providing the `shippingAddress`, those values will be kept in state.
- When the checkbox is set back to `false`.
  - The `shippingAddress` form group and all its form controls will be removed from the form.
  - The validation status of the form will be recalculated without the `shippingAddress` and the validators that were attached to it.

## But what if we want to keep the state in the form?

We generally would advise not to do that, but I will show you a few things you can try.

### Keeping the state of an input in the form

For this example we are going to add a `gender` property and a `genderOther` property.
When the `gender` (which is a radio button group) is set to `other` then, the `genderOther` control is added.

The code at this moment looks like this:

```typescript
export class MyFormComponent {
  ...

  protected readonly viewModel = computed(() => ({
    ...
    showOtherGender: this.formValue().gender === 'other'
  }));
  ...
}
```

```html
<div>
  <label>
    <span>Gender</span>
    Male
    <input
      type="radio"
      [ngModel]="vm.formValue.gender"
      name="gender"
      value="male"
    />
    Female
    <input
      type="radio"
      [ngModel]="vm.formValue.gender"
      name="gender"
      value="female"
    />
    Other
    <input
      type="radio"
      [ngModel]="vm.formValue.gender"
      name="gender"
      value="other"
    />
  </label>
</div>
<div>
  <label>
    <input
      type="text"
      [ngModel]="vm.formValue.genderOther"
      name="genderOther"
      *ngIf="vm.showOtherGender"
    />
  </label>
</div>
```

We calculate whether the `genderOther` input should be shown and in the HTML we bind 3 radio buttons to the `gender` property of our `formValue`.
We use an `*ngIf` directive to hide the `genderOther` based on the `showOtherGender` and that's it.
Now when we select `other` in the `gender` radiobutton and we type a value in the `genderOther`, the value will be lost when we switch it back to `female` or `male`.
The goal here is to make sure that the value of `genderOther` is maintained in the form, even when the value of `gender` is not set to `other`.

We can use the `[hidden]` functionality of Angular and make sure the item is hidden with `display: none` behind the scenes:

```html
<div>
  <label>
    <input type="text" ... name="genderOther" [hidden]="!vm.showOtherGender" />
  </label>
</div>
```

This would work but it might be more semantically correct to use a hidden field. For this we can use `input type="hidden"` and conditionally set the input `type` based on the result of the ViewModel:

```html
<div>
  <label>
    <input
      ...
      name="genderOther"
      [attr.type]="vm.showOtherGender? 'text': 'hidden'"
    />
  </label>
</div>
```

### Hiding an entire block

Having a conditional input type might result in more boilerplate if we want to hide an entire group for instance. You could also use a hidden field to bind an entire form group to. We can do this by using the combination of a hidden field with `[ngModel]`:

```html
<div ngModelGroup="addresses">
  <h3>Billing address</h3>
  <app-address ...></app-address>
  <input ... name="shippingAddressDifferentFromBillingAddress" />
  <div *ngIf="vm.showShippingAddress; else shippingAddressState">
    <h3>Shipping address</h3>
    <app-address ngModelGroup="shippingAddress" [address]="vm.shippingAddress">
    </app-address>
  </div>
  <!-- if it should be hidden
  bind the ormValue().addresses?.shippingAddress
  to the [ngModel]
  -->
  <ng-template #shippingAddressState>
    <input
      type="hidden"
      [ngModel]="vm.formValue.addresses?.shippingAddress"
      name="shippingAddress"
    />
  </ng-template>
</div>
```

## Wrap up

Check out the [Complete Stackblitz solution](https://stackblitz.com/edit/stackblitz-starters-c2bnme?file=src%2Fcomponents%2Fmy-form%2Fmy-form.component.html){:target="_blank"}

We learned that Template-driven Forms do all the work for us but also automatically
remove the values of the removed controls from our form state. In some cases we don't want that and we can work with a separate signal that holds the state value and calculate the computed value in the ViewModel.

We could also use a `[hidden]` directive from Angular to hide an input from the DOM but it might be more semantically correct to use a hidden field.

After that we saw how we can bind an entire form group to a hidden field to keep our state.
I hope you enjoyed this article! Stay tuned for more content!
[Interested in the YouTube video? Check it here](https://youtu.be/djod9on45wc){:target="_blank"}

**Update** A full working solution with a complex Demo can be found [here](https://blog.simplified.courses/i-opensourced-my-angular-template-driven-forms-solution/){:target="_blank"}


Special thanks to the reviewer:
- [Gerome Grignon](https://twitter.com/GeromeDEV){:target="_blank"}
