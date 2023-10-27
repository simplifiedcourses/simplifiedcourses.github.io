---
layout: post
title: "Angular Template-driven Forms with Signals"
date: 2023-10-12
published: true
cover: assets/template-driven-forms-with-signals.jpg
comments: true
categories: [ Angular, Forms, Signals, ngx-signal-state ]
description: "In this article we will create a type-safe Template-driven From with Signals with no boilerplate!"
---

In this article we will learn the most basic example of a semi-complex unidirectional form with Angular Signals.

Not a reader? I created 2 YouTube video's for you explaining everything!!
- [Livecode: Angular Unidirectional Template-driven Forms with signals](https://www.youtube.com/watch?v=ijp_qt3SYl4&t=1s){:target="_blank"}
- [Livecode: Angular Template-driven Forms with ViewModels](https://www.youtube.com/watch?v=ONOtrl6j6Qs&t=14s){:target="_blank"}

## Creating a unidirectional Template-driven form with Angular

Unidirectional dataflow is important for predictability, so we will not use the **banana-in-the-box** syntax `[()]`, but we will use `[ngModel]` without
the parenthesis. We will create a `form` element with 2 fields:
- `firstName`
- `lastName`

The HTML will look like this:

```html

<form>
    <label>
        <span>First name</span>
        <input type="text"
               [ngModel]="formValue().firstName"
               name="firstName" />
    </label>
    <label>
        <span>Last name</span>
        <input type="text"
               [ngModel]="formValue().lastName"
               name="lastName" />
    </label>
    <button type="submit">Submit</button>
</form>
```

Let's continue with the Typescript part and create a typed `MyFormModel` and add a `formValue` signal to the `MyFormComponent`:
```typescript
// We want our form to be typed
export type MyFormModel = {
    firstName: string;
    lastName: string;
};

@Component({
    ...
    imports: [FormsModule]
})
export class MyFormComponent {
    // Signal that holds the form value
    // Will be read by [ngModel]
    protected readonly formValue = signal<MyFormModel>({});
}
```

Somehow we need to update the `formValue` in a unidirectional way by listening to the changes of the form.
For that we can keep a reference to the `ngForm` directive (which is automatically added the to `form` tag when importing `FormsModule`) like this:

```html

<form #form="ngForm">
    ...
</form>
```

Now, by using a combination of `@ViewChild()` and the `afterViewInit()` lifecycle hook we can listen to the changes of the 
Reactive Form (that was automatically created by Angular) and feed the `formValue` signal.
```typescript
@Component({ ... })
export class MyFormComponent implements NgAfterViewInit {
    // Get access to the form
    @ViewChild('form') protected ngForm: NgForm | undefined;
    // Signal that holds the form value
    // Will be read by [ngModel]
    protected readonly formValue = signal<MyFormModel>({});
    
    public afterViewInit():void {
        // Listen to changes
        this.ngForm!.form.valueChanges.subscrige(v => {
            // Feed the signal ==> unidirectional dataflow
            this.formValue.set(v);
        })
    }
}
```

Great! The data from `formValue` will flow into our template, and all the form changes will be pushed back into the signal when the `valueChanges` 
of the form triggers... We achieved a unidirectional dataflow!

## Remove boilerplate

### Form directive

Let's remove the boilerplate by creating a `form` directive. This standalone directive will listen to the `form` DOM tag. 
It will inject `ngForm` and provide a `formValueChange` output.

```typescript
@Directive({
    selector: 'form',
    standalone: true,
})
export class FormDirective<T> {
    // Inject its own `NgForm` instance
    private readonly ngForm = inject(NgForm, { self: true });
    // Use the valueChanges of the form as the output
    @Output() public readonly formValueChange = this.ngForm.form.valueChanges;
}
```

At this moment in time the view is initialized so the `ngForm` already exists. This means we don't need the `afterViewInit` lifecycle hook.
The `formValueChange` will emit every time the form will update.

### Use the new Form Directive

We can start by importing the `FormDirective` and removing the `afterViewInit` lifecycle hook in our `MyFormComponent`.
We also don't need access to `ngForm` anymore.
In the HTML we can drop the `#form="ngForm"` and use the `(formValueChange)` output.
Below we see the entire HTML and Typescript of our Template-driven typesafe unidirectional Signal form.

```typescript
@Component({ 
    imports: [ FormsModule, FormDirective ]
})
export class MyFormComponent {
    // Signal that holds the form value
    protected readonly formValue = signal<MyFormModel>({});
}
```

```html
<form (formValueChange)="formValue.set($event)">
    <label>
        <span>First name</span>
        <input type="text"
               [ngModel]="formValue().firstName"
               name="firstName"
        />
    </label>
    <label>
        <span>Last name</span>
        <input type="text"
               [ngModel]="formValue().lastName"
               name="lastName"
        />
    </label>
    <button type="submit">Submit</button>
</form>
```

## A more complex example

We have a simple form with 2 values now, but let's create a more complex purchase form.
The form model looks like this:

```typescript
// address.form-model.ts
export type AddressFormModel = Partial<{
    street: string;
    number: string;
    city: string;
    zipcode: string;
    country: string;
}>

// purchase.form-model.ts
import { AddressFormModel } from './address.form-model';

export type PurchaseFormModel = Partial<{
    firstName: string;
    lastName: string;
    gender: 'male' | 'female' | 'other';
    genderOther?: string;
    age: number;
    emergencyContactNumber: string;
    addresses: Partial<{
        billingAddress: AddressFormModel;
        shippingAddress: AddressFormModel;
        shippingAddressDifferentFromBillingAddress: boolean;
    }>;
}>;
```

Let's start by creating the DOM let's use the `[ngModel]` and `ngModelGroup` to build our entire form:
`firstName`, `lastName`, `age`, `emergencyContactNumber`, `gender`, `genderOther` all have `[ngModel]` directive and
`name` attributes and we have an `ngModelGroup` called `addresses` that has 2 more child `ngModelGroup` instances for
`billingAddress` and `shippingAddress`. Those will use the `AddressFormComponent`:

```html

<form ...>
    <div>
        <label>
            <span>First name</span>
            <input type="text"
                   [ngModel]="vm.formValue.firstName"
                   name="firstName"/>
        </label>
    </div>
    <div> ... lastName</div>
    <div> ... age</div>
    <div> ...emergencyContactNumber</div>
    <div> ... gender</div>
    <div> ... genderOther</div>
    <div ngModelGroup="addresses">
        <div ngModelGroup="billingAddress">
            <app-address-form [address]="vm.formValue.addresses?.billingAddress">
            </app-address-form>
        </div>
        <label> ...shippingAddressDifferentFromBillingAddress </label>
        <div ngModelGroup="shippingAddress">
            <app-address-form [address]="vm.formValue.addresses?.shippingAddress">
            </app-address-form>
        </div>
    </div>
    <button type="submit">Submit</button>
</form>
```

The typescript part looks straight forward for now. As we can see we have created a ViewModel that exposes
the value of the `formValue` signal to the template through a getter `vm`:

```typescript
@Component({
    ...
    imports: [
        FormsModule,
        FormDirective,
        AddressFormComponent,
        ...
    ]
   ...
})
export class MyFormComponent {
    protected readonly formValue = signal<PurchaseFormModel>({});

    private readonly viewModel = computed(() => ({
        formValue: this.formValue(),
    }));

    protected get vm() {
        return this.viewModel();
    }
    ...
}
```

### Template-driven form ViewModel logic

There are some rules for this form:
- The `emergencyContactNumber`  control has to be disabled when the age is `0` or higher than `17`.
- The `genderOther` field should only be there when the `gender` its value is set to `other`.
- The `shippingAddress` group should only be there when the `shpipingAddressDifferentFromBillingAddress` is set to `true`.

This setup for forms makes this logic very easy and declarative.
The only thing we need to do is update the ViewModel like this:

```typescript
export class MyFormComponent {
    protected readonly formValue = signal<PurchaseFormModel>({});

    private readonly viewModel = computed(() => ({
        formValue: this.formValue(),
        emergencyContactDisabled:
            this.formValue().age === 0 || this.formValue().age >= 18,
        showOtherGender: this.formValue().gender === 'other',
        showShippingAddress: 
            this.formValue().addresses?.shippingAddressDifferentFromBillingAddress,
    }));

    protected get vm() {
        return this.viewModel();
    }
    ...
}
```

This puts all that logic into one declarative readable ViewModel that we can expose to the template through the `vm` getter.

Now the only thing we need to do is use the `*ngIf` directive and `[disabled]` input to make Angular react to our ViewModel:

```html

<form ...>
    <div> ... firstName</div>
    <div> ... lastName</div>
    <div> ... age</div>
    <div>
        <label>
            <span>Emergency contact</span>
            <input ...
                   name="emergencyContactNumber"
                   [disabled]="vm.emergencyContactDisabled"
            />
        </label>
    </div>
    <div> ... gender</div>
    <div>
        <label>
            <input ...
                   name="genderOther"
                   *ngIf="vm.showOtherGender"
            />
        </label>
    </div>
    <div ngModelGroup="addresses">
        <div ngModelGroup="billingAddress"> ...</div>
        <label> ...shippingAddressDifferentFromBillingAddress </label>
        <div *ngIf="vm.showShippingAddress"
             ngModelGroup="shippingAddress"> .../div>
        </div>
        <button type="submit">Submit</button>
</form>
```

## Conclusion

- Template-driven Forms can be very easy and we can build a lot with very limited code.
- Angular creates reactive form controls and form groups for us behind the scenes. We let Angular do all the hard work for us.
- We can avoid using `ngForm` and the `ngAfterViewInit` lifecycle hook by using a custom `FormDirective`.
- We can leverage a ViewModel and send a tailored declarative object to our template and bind it to `*ngIf` directives and `[disabled]` inputs.
- Everything is typesafe but the `name` attribute.

[Checkout the Stackblitz example here](https://stackblitz.com/edit/stackblitz-starters-og2pnw){:target="_blank"}

This example does not contain Model validations, conditional logic, and asynchronous logic because they are out of scope for this article.
If you reach out I will gladly share examples.
Also read [Say goodbye to custom form validators in Angular](https://blog.simplified.courses/say-goodbye-to-custom-form-validators-in-angular/){:target="_blank"}
and [Template-driven forms with form arrays in Angular](https://blog.simplified.courses/template-driven-forms-with-form-arrays/){:target="_blank"}

For more complex forms, either send me a DM or wait for new articles. I hope you enjoyed it!


