---
layout: post
title: "Template-driven forms with form arrays in Angular"
date: 2023-08-06
published: true
comments: true
categories: [ Angular, Forms, State management, Signals ]
cover: assets/template-driven-forms-with-form-arrays-in-angular.jpg
description: "This article explains how to work with form arrays in template-driven forms in Angular"
---

## Intro

**Update 17-10-2023: This article is been updated for Angular Signals and does not use ObservableState anymore**

Lately, more and more Angular developers favor template-driven forms over reactive forms because:

- There is almost no boilerplate code.
- It's easy to
  do [form validations](https://blog.simplified.courses/say-goodbye-to-custom-form-validators-in-angular/){:target="_
  blank"}.
- We let Angular do all the work! Angular will create `FormControl` and `FormGroup` instances for us automatically
  behind the scenes.
- Template-driven forms are more declarative.
  The pros and cons of both techniques are explained
  in [this article](https://blog.simplified.courses/template-driven-or-reactive-forms-in-angular/){:target="_blank"}.

When using Reactive forms in Angular we could use a `FormArray` if we want to create an iterable list of `FormControl`
elements.
This technique can be handy if we want to create a list of phonenumbers for instance.
**Template-driven forms don't play nice with form arrays** and in this article we will focus on how to achieve the same
functionality by using template-driven forms.
We will create a form with phonenumbers to showcase how we can achieve the same functionality with template-driven
forms.
To be ahead of the game this article uses Angular Signals behind the scenes. Check this article if you want to know how
to set up a [Template-driven form with Signals](https://simplified.courses/template-driven-forms-with-form-arrays/){:
target="_blank"}

Like we mentioned before, using a `FormArray` in combination with template-driven forms is not possible. The `FormArray`
class belongs to the reactive forms package and template-driven forms can only
create `FormGroup` and `FormControls` instances automatically by using the `ngModel` and `ngModelGroup` directives.

## The phonenumbers functionality

Let's continue with our phonenumbers example. We need a simple user form that has a `firstName` and `lastName` property,
together with an array of phonenumbers.
FormArrays are not available in template-driven forms, but we could use the following technique to achieve something
similar:

```html

<form ...>
    <label>
        First name
        <input type="text" [ngModel]="vm.user.firstName" name="firstName"/>
    </label>
    <label>
        Last name
        <input type="text" [ngModel]="vm.user.lastName" name="lastName"/>
    </label>
    <div *ngFor="let phoneNumber of phoneNumbers; let i = index">
        <input type="text" [ngModel]="phoneNumbers[i]" name="phoneNumber{{i}}"/>
    </div>
    ...
</form>
```

This would create a `FormControl` instance on the `form` for every phonenumber and **would pollute the automatically
created reactive form** behind the scenes. It would look like this:

```typescript
ngForm = {
    form: {
        firstName: FormControl,
        lastName: FormControl,
        phonenumber0: FormControl, // Dirty
        phonenumber1: FormControl, // Dirty
        phonenumber2: FormControl, // Dirty
        phonenumber3: FormControl, // Dirty
    }
}
```

As we can see, an array is not possible. Angular would just create formControls with unique keys.
Let's stop trying to use arrays, and instead use template-driven forms they way they were meant to be used.
This means our form can only exist out of `FormGroup` instances and `FormControl` intances.
The previous approach polluted the structure of our form. The following structure is the structure that we want to
achieve:

```typescript
ngForm = {
    form: {
        firstName: FormControl,
        lastName: FormControl,
        // Clean separate formGroup that contains the phonenumbers
        // on a property related to the index
        phonenumbers: {
            0: FormControl,
            1: FormControl,
            2: FormControl,
            3: FormControl,
        }
    }
}
```

We can see that the form is not polluted anymore and we use a `phonenumbers` object with indexes as keys that will
contain the `FormControl` instances for every phonenumber.

Let's update the `UserFormModel` since this is the model that will represent our actual form:

```typescript
type UserFormModel = Partial<{
    firstName: string;
    lastName: string;
    // Will represent the input that will be used
    // to add a phone number
    addPhonenumber: string;
    // The list of actual phone numbers
    phonenumbers: { [key: string]: string };
    ...
}>
```

Ideally, an initial version of our form would look like this:
**Note: always use a trackBy function**

```html

<form ...>
    ...
    <div ngModelGroup="phonenumbers">
        <div *ngFor="let key of vm.user.phonenumbers; trackBy: tracker">
            <input type="text"
                   [ngModel]="vm.user.phonenumbers[key]"
                   name="{%raw%}{{key}}{%endraw%}"/>
        </div>
    </div>
    ...
</form>
```

```typescript
export class UserFormComponent implements AfterViewInit {
    @ViewChild('form') form!: NgForm;
    private readonly user = signal<UserFormModel>({});

    private readonly viewModel = computed(() => ({
        user: this.user()
    }))

    protected get vm() {
        return this.viewModel();
    };

    protected tracker = (i: number) => i;

    public ngAfterViewInit(): void {
        this.form.valueChanges?.subscribe((v) => {
            this.user.set(v);
        });
    }
```

**This will result in an error** because `vm.user.phonenumbers` **is not an iterable but an object** and the `*ngFor`
directive expects an iterable like an Array.
To convert this object to an array we can use the `keyvalue` pipe. This pipe will return an array with a `key` and
a `value` property.

```html

<form ...>
    ...
    <div ngModelGroup="phonenumbers">
        <div *ngFor="let item of vm.user.phonenumbers | keyvalue; trackBy: tracker">
            <input type="text"
                   [ngModel]="vm.user.phonenumbers[item.key]"
                   name="{%raw%}{{item.key}}{%endraw%}"/>
        </div>
    </div>
    ...
</form>
```

We can see that we use the key (index) in the `name` attribute.
We also use the `key` to bind the actual phonenumber to the `[ngModel]` directive. Note that we do not use the banana in
the box syntax because `this.form.valueChanges` is automatically feeding our component its state machine in
the `ngAfterViewInit()`
lifecycle hook.

## Completing the phonenumbers functionality with adding and deleting phonenumbers

The iterative part and the updating of phonenumbers is ready, but now we still need to be able to add and remove
phonenumbers.

```html

<form ...>
    ...
    <div ngModelGroup="phonenumbers">
        <div *ngFor="let item of vm.user.phonenumbers | keyvalue; trackBy: tracker">
            <input type="text"
                   *ngIf="vm.user.phonenumbers as phonenumbers"
                   [ngModel]="phonenumbers[item.key]"
                   name="{%raw%}{{item.key}}{%endraw%}"/>

            <!-- Delete the phonenumber based on the key -->
            <button type="button" (click)="deletePhonenumber(item.key)">
                Delete phonenumber
            </button>
        </div>
    </div>
    <!-- Bind the addPhonenumber to an input -->
    <input type="text" [ngModel]="vm.user.addPhonenumber" name="addPhonenumber"/>
    <!-- Add a phonenumber -->
    <button type="button" (click)="addPhonenumber()">Add phonenumber</button>
    ...
</form>
```

The template is ready, we only need to add a `deletePhonenumber()` and `addPhoneNumber()` method.
An array would be a little bit easier here, but since template-driven forms don't play nice with arrays we have to
translate an array to an object and the other way around.

Let's start with the `addPhonenumber()` method. We will use `Object.values` to get all the values from our
phonenumbers object and create a new array with the `addPhonenumber` value. This gives us a brand-new array with the
newly added phonenumber in it:

```typescript
export class UserFormComponent implements AfterViewInit {
...
    protected addPhonenumber(): void {
        // Create new array with all the old phonenumbers and new one
        const phonenumbers = [
            ...Object.values(this.user().phonenumbers),
            this.user().addPhonenumber,
        ];
    }
}
```

What's left to do is update our `user` ViewModel with the newly calculated phonenumbers. For that we need to convert our
new array back to an object where all the keys are indexes. We can clean the `addPhonenumber` field at the same time.

```typescript
export class UserFormComponent implements AfterViewInit {
...

    protected addPhonenumber(): void {
        // Create new array with all the old phonenumbers and new one
        const phonenumbers = [
            ...Object.values(this.user().phonenumbers || {}),
            this.user().addPhonenumber,
        ] as string[];
        this.user.update((old) => ({
            ...old,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        }));
    }
}
```

To create an object from an array we can use the `reduce()` method that lives on the arrays prototype. We created a
simple pure `arrayToObject` function that can be reused everywhere:

```typescript
// ['foo', 'bar', 'baz'] => {0: 'foo', 1: 'bar', 2: 'baz'}
function arrayToObject<T>(arr: T[]): { [key: number]: T } {
    return arr.reduce((acc, value, index) => ({ ...acc, [index]: value }), {})
}
```

For the `deletePhonenumber()` method we can use `Object.values` to get all the values from our phonenumbers object and
use the `filter` method to make sure that the phonenumber we want to delete isn't part of the array anymore.

```typescript
export class UserFormComponent implements AfterViewInit {
...

    protected deletePhonenumber(key: string): void {
        const phonenumbers = Object
            .values(this.user().phonenumbers)
            .filter(
                (v, index) => index !== Number(key)
            );
    }
}
```

Updating is done exactly the same as we did with the `addPhoneNumber()` method. Below you can see both methods
implemented:

```typescript
export class UserFormComponent implements AfterViewInit {
...

    protected addPhonenumber(): void {
        const phonenumbers = [
            ...Object.values(this.user().phonenumbers),
            this.user().addPhonenumber,
        ];
        this.user.update((old) => ({
            ...old,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        }));
    }

    public deletePhonenumber(key: string): void {
        const phonenumbers = Object.values(this.user().phonenumbers || {}).filter(
            (v, index) => index !== Number(key)
        );
        this.user.update((old) => ({
            ...old,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        }));
    }
}

```

The outcome of our form (what is being kept in the state of our component now looks like this:

```typescript
form = {
    user: {
        firstName: "Brecht",
        lastName: "Billiet",
        addPhonenumber: "",
        phonenumbers: {
            0: "000000000000",
            1: "111111111111",
            2: "222222222222"
        }
    }
}
```

Here is an overview of the entire code of `user-form.component.ts`:

```typescript
import { CommonModule } from '@angular/common';
import { AfterViewInit, Component, signal, ViewChild } from '@angular/core';
import { FormsModule, NgForm } from '@angular/forms';

type UserFormModel = Partial<{
    firstName: string;
    lastName: string;
    addPhonenumber: string;
    phonenumbers: { [key: string]: string };
}>;

@Component({
    selector: 'app-user-form',
    templateUrl: './user-form.component.html',
    styleUrls: ['./user-form.component.css'],
    standalone: true,
    imports: [CommonModule, FormsModule],
})
export class UserFormComponent implements AfterViewInit {
    @ViewChild('form') form!: NgForm;
    private readonly user = signal<UserFormModel>({});

    protected get vm() {
        return { user: this.user() };
    }

    tracker = (i: number) => i;

    public ngAfterViewInit(): void {
        this.form.valueChanges?.subscribe((v) => {
            this.user.set(v);
        });
    }

    protected submit(): void {
        console.log(this.form);
    }

    protected addPhonenumber(): void {
        const phonenumbers = [
            ...Object.values(this.user().phonenumbers || {}),
            this.user().addPhonenumber,
        ] as string[];
        this.user.update((old) => ({
            ...old,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        }));
    }

    protected deletePhonenumber(key: string): void {
        const phonenumbers = Object.values(this.user().phonenumbers || {}).filter(
            (v, index) => index !== Number(key)
        );
        this.user.update((old) => ({
            ...old,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        }));
    }
}

function arrayToObject<T>(arr: T[]): { [key: number]: T } {
    return arr.reduce((acc, value, index) => ({ ...acc, [index]: value }), {});
}
```

The complete result of the template looks like this. This template has no boilerplate, it is readable and Angular takes
care of everything for us:

```html

<form #form="ngForm" (ngSubmit)="submit()">
    <label>
        First name
        <input type="text" [ngModel]="vm.user.firstName" name="firstName"/>
    </label>
    <label>
        Last name
        <input type="text" [ngModel]="vm.user.lastName" name="lastName"/>
    </label>
    <h2>Phonenumbers</h2>
    <div ngModelGroup="phonenumbers">
        <div class="phonenumber"
             *ngFor="let item of vm.user.phonenumbers | keyvalue; trackBy: tracker">
            <input type="text"
                   *ngIf="vm.user.phonenumbers as phonenumbers"
                   [ngModel]="phonenumbers[item.key]"
                   name="{{ item.key }}"/>
            <button type="button" (click)="deletePhonenumber(item.key)">
                Delete phonenumber
            </button>
        </div>
    </div>
    <input type="text" [ngModel]="vm.user.addPhonenumber" name="addPhonenumber"/>
    <button type="button" (click)="addPhonenumber()">Add phonenumber</button>
    <button>Submit form</button>
</form>

```

You can play with the example
on [Stackblitz here](https://stackblitz.com/edit/angular-ngwvfw?file=src%2Fuser-form%2Fuser-form.component.html){:
target="_blank"}.

### Conclusion

Template-driven forms are clean, have no boilerplate and we let Angular take care of all the hard work for us.
We can not use `FormArray` since it is a part of the reactive forms package in Angular. Since template-driven forms
create `FormControl` and `FormGroup` instances
for us automatically we can create clean form structures by converting arrays to objects and the other way around.
Hope you liked the article! If you have any questions! Reach out!

Remember that we also offer [Angular Consultancy](https://www.simplified.courses/angular-coaching){:target="_blank"}
and [Angular Coaching](https://www.simplified.courses/angular-coaching){:target="_blank"}. We also love to help out
with [Angular Training](https://www.simplified.courses/angular-training){:target="_blank"} 