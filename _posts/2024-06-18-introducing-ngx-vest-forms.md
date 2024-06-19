---
layout: post
title: "Introducing ngx-vest-forms: Simplify Complex Angular Forms"
date: 2024-06-18
published: true
cover: assets/introducing-ngx-vest-forms.jpg
comments: false
authors: [brecht_billiet]
categories: [Angular forms]
description: "Learn how ngx-vest-forms simplifies the creation of complex Angular template-driven forms with robust validation using vest.js."
---

### Introduction

Over a year ago, I started investigating template-driven forms to find a more efficient way to handle complex validations and conditionals.
This led to the development of ngx-vest-forms, a lightweight adapter for Angular template-driven forms integrated with [vest.js](https://vestjs.dev){:target="_blank"}. 
The goal was to create unidirectional forms without any boilerplate code. 
This solution is now battle-tested on large projects, and I’ve also created a course to help others master it.
I even published an npm package called [ngx-vest-forms](https://www.npmjs.com/package/ngx-vest-forms){:target="_blank"}.
My solution has evolved a lot over time and since Angular 18 released a new `events` obserfable I was able to clean it up even more.

Some characteristics of **ngx-vest-forms**
- 🚀 **Lightweight and Efficient**: ngx-vest-forms is a very lightweight adapter, ensuring your applications remain fast and responsive.
- 🔄 **Unidirectional Data Flow**: Promotes clean architecture with unidirectional data flow, reducing complexity and potential bugs.
- ✅ **Asynchronous Validations**: All validations are asynchronous, providing the ability to interact with the backend.
- 💼 **Battle-Tested**: The solution is battle-tested on big projects, ensuring reliability and performance.
- 🔍 **Type Safety**: Type-safe template-driven forms help avoid runtime errors and typos, enhancing development productivity.
- 🛠️ **Reusability**: Validation suites can be reused across different frameworks and technologies, resulting in code reusability and maintainability.
- 🔄 **Reactive Disabling**: Easily implement reactive disabling of form fields based on dynamic conditions, for better user interactions.
- 🧩 **Composable Validations**: Allows composing validation suites for better readability and reusability of validation logic.
- 🧪 **Robust Testing**: Built-in support for vest.js makes it easy to write and maintain robust validation tests.
- 📚 **Comprehensive Course**: A dedicated course is available to help you master complex Angular template-driven forms quickly.

Oh, the package also contains a complex demo:
[Check it out here](https://github.com/simplifiedcourses/ngx-vest-forms/tree/master/projects/examples){:target="_blank"}

### Installation

You can install ngx-vest-forms by running:

```shell
npm i ngx-vest-forms
```
### Creating a Simple Form
To create a simple form with **ngx-vest-forms**, follow these steps. 
We’ll create a form group called `generalInfo` with two properties: `firstName` and `lastName`.

First, import the `vestForms` const in the imports section of the @Component decorator. 
Then, apply the scVestForm directive to the `form` tag and listen to the `formValueChange` output to feed a signal.

```typescript
import { vestForms, DeepPartial } from 'ngx-vest-forms';

// We want our form to be type-safe
type MyFormModel = DeepPartial<{
  generalInfo: {
    firstName: string;
    lastName: string;
  }
}>

@Component({
  imports: [vestForms],
  template: `
<form scVestForm 
      (formValueChange)="formValue.set($event)"
      (ngSubmit)="onSubmit()">
      <div ngModelGroup="generalInfo">
        <label>First name</label>
        <input type="text" name="firstName" [ngModel]="formValue().generalInformation?.firstName"/>
      
        <label>Last name</label>
        <input type="text" name="lastName" [ngModel]="formValue().generalInformation?.lastName"/>
      </div>
</form>
  `
})
export class MyComponent {
  protected readonly formValue = signal<MyFormModel>({});
}
```

We can see that we are using `ngModel` and `ngModelGroup` directives in combination with the `name` attribute.
We also notice that **we don't use the banana-in-the-box syntax**. This results in a unidirectional dataflow.

The first step of our form is done.

### Avoiding Typo Errors

Template-driven forms are type-safe but not in the `name` or `ngModelGroup` attributes. 
To avoid typos, we introduced shapes. A shape is a deep required version of the form model used for validation.

**ngx-vest-forms** exports some handy opinionated types to make types deep partial or deep required:
In this example we create a shape, and pass it to the `formShape` input of the `scVestForm` directive.

```typescript
import { DeepPartial, DeepRequired, vestForms } from 'ngx-vest-forms';

type MyFormModel = DeepPartial<{
  generalInfo: {
    firstName: string;
    lastName: string;
  }
}>

export const myFormModelShape: DeepRequired<MyFormModel> = {
  generalInfo: {
    firstName: '',
    lastName: '' 
  }
};

@Component({
  imports: [vestForms],
  template: `
<form scVestForm 
      [formShape]="shape"
      (formValueChange)="formValue.set($event)"
      (ngSubmit)="onSubmit()">
      
      <div ngModelGroup="generalInfo">
        <label>First name</label>
        <input type="text" name="firstName" [ngModel]="formValue().generalInformation?.firstName"/>
      
        <label>Last name</label>
        <input type="text" name="lastName" [ngModel]="formValue().generalInformation?.lastName"/>
      </div>
</form>
  `
})
export class MyComponent {
  protected readonly formValue = signal<MyFormModel>({});
  protected readonly shape = myFormModelShape;
}

```
Or form is completely typo-safe now. In development the developer will get a runtime error with all the information he or she needs.
For DX the error looks like this, pinpointing where the developer messed up:


```chatinput
Error: Shape mismatch:

[ngModel] Mismatch 'firstame'
[ngModelGroup] Mismatch: 'addresses.billingddress'
[ngModel] Mismatch 'addresses.billingddress.steet'
[ngModel] Mismatch 'addresses.billingddress.number'
[ngModel] Mismatch 'addresses.billingddress.city'
[ngModel] Mismatch 'addresses.billingddress.zipcode'
[ngModel] Mismatch 'addresses.billingddress.country'


    at validateShape (shape-validation.ts:28:19)
    at Object.next (form.directive.ts:178:17)
    at ConsumerObserver.next (Subscriber.js:91:33)
    at SafeSubscriber._next (Subscriber.js:60:26)
    at SafeSubscriber.next (Subscriber.js:31:18)
    at subscribe.innerSubscriber (switchMap.js:14:144)
    at OperatorSubscriber._next (OperatorSubscriber.js:13:21)
    at OperatorSubscriber.next (Subscriber.js:31:18)
    at map.js:7:24
```

### Conditional fields and Conditional disabling

**ngx-vest-forms** is just a box of a couple of directives helping you to achieve better validations and
handy outputs. It's just Angular really. So what I'm showing here does not have anything to do with **ngx-vest-forms**.
It has to do with the awesomeness of template-driven forms.

To hide a field based on the value of another, use computed signals. For instance, hide lastName if firstName is not filled in:

```html
<div ngModelGroup="generalInfo">
  <label>First name</label>
  <input type="text" name="firstName" [ngModel]="formValue().generalInformation?.firstName"/>
  
  @if(lastNameAvailable()){
    <label>Last name</label>
    <input type="text" name="lastName" [ngModel]="formValue().generalInformation?.lastName"/>
  }
</div>
```

```typescript
class MyComponent {
  ...
  protected readonly lastNameAvailable = 
    computed(() => !!this.formValue().generalInformation?.firstName);
}
```

The same goes for disabling.

```html
<input type="text" name="lastName" 
       [disabled]="lastNameDisabled()" 
       [ngModel]="formValue().generalInformation?.lastName"/>
```

```typescript
class MyComponent {
  protected readonly lastNameDisabled = 
    computed(() => !this.formValue().generalInformation?.firstName);
}
```

### Validations

I love template-driven forms for a while now, but I always struggled with validations.
When I saw some talks of Ward Bell, he introduced **vest.js**, an awesome validation framework.
After implementing it in real big projects with complex forms, I created **ngx-vest-forms** (the adaptor)

#### How does it work?

It's actually ridiculously simple. I wrote 3 directives.
- `scVestForm`: Which holds the formValue, a validation suite and some handy outputs
- `formModelDirective`: Yes... Hooks into the `ngModel` selector and creates an async angular validator for us behind the scenes.
- `formModelGroupDirective`: Hooks into the `ngModelGroup` selector and creates an angular async validator for us behind the scenes.

I also wrote a component that does content projection
- `sc-control-wrapper`: This just shows the validation errors in a consistent way.

So every time a control or control group is generated, the `formModelDirective` and `formModelGroupDirectives` will create real
Angular validators that just use a vest suite to use complex validations. After that it will render the errors in the `sc-control-wrapper` layout
component, both for form controls and form groups

Easy peasy, lemon squeezy!

#### Setting up our first validation suite

```typescript
import { enforce, only, staticSuite, test } from 'vest';
import { MyFormModel } from '../models/my-form.model'

export const myFormModelSuite = staticSuite(
    (model: MyformModel, field?: string) => {
      if (field) {
        only(field);
      }
      test('firstName', 'First name is required', () => {
        enforce(model.firstName).isNotBlank();
      });
      test('lastName', 'Last name is required', () => {
        enforce(model.lastName).isNotBlank();
      });
    }
);
```

After that we connect that suite to our form:

```typescript
class MyComponent {
  protected readonly formValue = signal<MyFormModel>({});
  protected readonly suite = myFormModelSuite;
}
```

```html
<form scVestForm
      [formShape]="shape"
      [formValue]="formValue"
      [suite]="suite"
      (formValueChange)="formValue.set($event)"
      (ngSubmit)="onSubmit()">
  ...
</form>

```

#### Showing the validation errors

Like mentioned before we have the `sc-control-wrapper` to show the actual errors:

```html
<div ngModelGroup="generalInfo" sc-control-wrapper>
  <div sc-control-wrapper>
    <label>First name</label>
    <input type="text" name="firstName" [ngModel]="formValue().generalInformation?.firstName"/>
  </div>

  <div sc-control-wrapper>
    <label>Last name</label>
    <input type="text" name="lastName" [ngModel]="formValue().generalInformation?.lastName"/>
  </div>
</div>
```

This removes tons of boilerplate code from our applications. We let Angular do all the work and it opens the door
to really complex forms and complex validations.

Some great features of vest validation suites:
- 🧪 **Declarative Syntax**: Vest suites use a declarative syntax that is easy to read and write, similar to unit tests.
- 🔄 **Reusable and Composable**: Validation logic can be modular and reusable across different parts of an application or even across different projects.
- ⚡ **Performance Optimization**: Vest allows for selective validation with the `only` function, running validations only for specific fields, which enhances performance.
- 📚 **Framework Agnostic**: Vest can be used with any frontend or backend framework, providing flexibility in choosing your technology stack.
- 🛠️ **Highly Configurable**: Supports complex validation scenarios with features like conditional validations and custom validation functions.
- 🚀 **Asynchronous Validation**: Built-in support for asynchronous validations helps in creating smooth and responsive forms.
- 🔍 **Detailed Error Reporting**: Provides comprehensive error messages and detailed reporting, making it easier to debug validation issues.
- 🔄 **Dynamic and Conditional Validation**: Easily handle dynamic forms and conditional validations, such as fields that are required only under certain conditions.
- 🌐 **Community and Documentation**: Vest has good community support and comprehensive documentation, which helps in getting started and troubleshooting issues.
- 🔄 **Integrated with ngx-vest-forms**: Seamless integration with ngx-vest-forms for Angular applications, providing a robust solution for form validation.

### More complex validations

#### Conditional
Vest makes conditional validations easy. For instance, emergencyContact is required only if the person is under 18:

```typescript
import { enforce, omitWhen, only, staticSuite, test } from 'vest';

...
omitWhen((model.age || 0) >= 18, () => {
  test('emergencyContact', 'Emergency contact is required', () => {
    enforce(model.emergencyContact).isNotBlank();
  });
});

```

#### Composable

You can compose validation suites for better reusability and readability. Here’s how to validate an address:

```typescript
export function addressValidations(model: AddressModel | undefined, field: string): void {
  test(`${field}.street`, 'Street is required', () => {
    enforce(model?.street).isNotBlank();
  });
  test(`${field}.city`, 'City is required', () => {
    enforce(model?.city).isNotBlank();
  });
  test(`${field}.zipcode`, 'Zipcode is required', () => {
    enforce(model?.zipcode).isNotBlank();
  });
  test(`${field}.number`, 'Number is required', () => {
    enforce(model?.number).isNotBlank();
  });
  test(`${field}.country`, 'Country is required', () => {
    enforce(model?.country).isNotBlank();
  });
}
```

They can be consumed in the parent suite:

```typescript
import { only, staticSuite } from 'vest';
import { PurchaseFormModel } from '../models/purchaseFormModel';

export const mySuite = staticSuite(
    (model: PurchaseFormModel, field?: string) => {
        if (field) {
            only(field);
        }
        addressValidations(model.addresses?.billingAddress, 'addresses.billingAddress');
        addressValidations(model.addresses?.shippingAddress, 'addresses.shippingAddress');
    }
);
```

### Async validations

We can use factory functions for vest suites. We can use that to pass a translation service to do internationalization
or to perform asynchronous validations.

Here is an example on how to do asynchronous validations with vest.

```typescript
export const createSimpleFormValidations = (swapiService: SwapiService) => {
    return staticSuite((model: SimpleFormModel, field: string) => {
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

I explain it more in depth in this [asynchronous form validation article](https://blog.simplified.courses/asynchronous-form-validators-in-angular-with-vest/){:target="_blank"}

### Validations on the root form

It is also possible to do validations on the root form.
When setting the [validateRootForm] directive to true, the form will also create an ngValidator on root level, that listens to the ROOT_FORM field.
In the example we set the `validateRootForm` directive and we keep our errors signal up to date
so we can consume them in the HTML.

```html
{{ errors()?.['rootForm'] }} <!-- render the errors on the rootForm -->
{{ errors() }} <!-- render all the errors -->
<form scVestForm 
      ...
      [validateRootForm]="true"
      (errorsChange)="errors.set($event)"
      ...>
</form>
```

```html
export class MyformComponent {
  protected readonly formValue = signal<MyFormModel>({});
  protected readonly suite = myFormModelSuite;
  // Keep the errors in state
  protected readonly errors = signal<Record<string, string>>({ });
}
```

```typescript
import { ROOT_FORM } from 'ngx-vest-forms';

export const mySuite = staticSuite(
    (model: PurchaseFormModel, field?: string) => {
        if (field) {
            only(field);
        }
        test(ROOT_FORM, 'Brecht is not 30 anymore', () => {
            enforce(
                model.firstName === 'Brecht' &&
                model.lastName === 'Billiet' &&
                model.age === 30).isFalsy();
        });
    });

```


### Validation of dependant controls and or groups

Sometimes, form validations are dependent on the values of other form controls or groups. 
This scenario is common when a field's validity relies on the input of another field. 
A typical example is the `confirmPassword` field, which should only be validated if the `password` field is filled in. 
When the `password` field value changes, it necessitates re-validating the `confirmPassword` field to ensure 
consistency.

Here's how you can handle validation dependencies with ngx-vest-forms and vest.js:


Use Vest to create a suite where you define the conditional validations. 
For example, the `confirmPassword` field should only be validated when the `password` field is not empty. 
Additionally, you need to ensure that both fields match.

```typescript
import { enforce, omitWhen, staticSuite, test } from 'vest';
import { MyFormModel } from '../models/my-form.model';

export const myFormModelSuite = staticSuite((model: MyFormModel, field?: string) => {
    if (field) {
        only(field);
    }

    test('password', 'Password is required', () => {
        enforce(model.password).isNotBlank();
    });

    omitWhen(!model.password, () => {
        test('confirmPassword', 'Confirm password is required', () => {
            enforce(model.confirmPassword).isNotBlank();
        });
    });

    omitWhen(!model.password || !model.confirmPassword, () => {
        test('passwords', 'Passwords do not match', () => {
            enforce(model.confirmPassword).equals(model.password);
        });
    });
});
```

Creating a validation config.
The `scVestForm` has an input called `validationConfig`, that we can use to let the system know when to retrigger validations.

```typescript
protected validationConfig = {
    password: ['passwords.confirmPassword']
}
```
Here we see that when password changes, it needs to update the field `passwords.confirmPassword`.
This validationConfig is completely dynamic, and can also be used for form arrays.

```html

<form scVestForm
      ...
      [validationConfig]="validationConfig">
    <div ngModelGroup="passwords">
        <label>Password</label>
        <input type="password" name="password" [ngModel]="formValue().passwords?.password"/>

        <label>Confirm Password</label>
        <input type="password" name="confirmPassword" [ngModel]="formValue().passwords?.confirmPassword"/>
    </div>
</form>
```

### Form arrays

If you want to know how to deal with form arrays, check out https://blog.simplified.courses/template-driven-forms-with-form-arrays/

### Wrap up

This package is a result of hard work and a lot of research. If you want to support me,
you can give a star on [GitHub](https://github.com/simplifiedcourses/ngx-vest-forms){:target="_blank"}, or collaborate with me.
Please check out the docs, and the example.

If you want to check the demo, clone the project locally, and run:

```shell
cd ngx-vest-forms
npm i
npm run ngx-vest-forms:build
npm start
```

It will spin up a beautifully designed complex form this is powered by tailwindCSS and flowbite.
[Here](https://stackblitz.com/~/github.com/simplifiedcourses/ngx-vest-forms-stackblitz){:target="_blank"} is a stackblitz example for you.
Enjoy being extremely productive in forms!!

