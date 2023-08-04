---
layout: post
title:  "Template-driven forms with form arrays in Angular"
date:   2023-07-30
published: true
comments: true
categories: [Angular, Forms, State management, Signals]
cover: assets/template-driven-forms-with-form-arrays-in-angular.jpg
description: "This article explains how to work with form arrays in template-driven forms in Angular"
---

## Intro

Lately, more and more Angular developers favor template-driven forms over reactive forms because:
- There is almost no boilerplate code.
- It's easy to do [form validations](https://blog.simplified.courses/say-goodbye-to-custom-form-validators-in-angular/){:target="_blank"}
- We let Angular do all the work! Angular will create `FormControl` and `FormGroup` instances for us automatically behind the scenes.
- Template-driven forms are more declarative.
I explain the pros and cons of both techniques in [this article](https://blog.simplified.courses/template-driven-or-reactive-forms-in-angular/){:target="_blank"}.

When using Reactive forms in Angular we could use a `FormArray` if we want to create an iterable list of `FormControl` elements. This technique can be handy if we want to create a list of phonenumbers for instance.
**Template-driven forms don't play nice with form arrays** and in this article we will focus on how to achieve the same functionality by using template-driven forms.
We will create a form with phonenumbers to showcase how we can achieve the same functionality with template-driven forms.

In the article mentioned before, I explain how we can achieve reactivity in template-driven forms by using [ObservableState](https://github.com/simplifiedcourses/observable-state){:target="_blank"} (a simple custom opinionated state management class). The code of a simple form looks like this:

```typescript
// Create a model for our form, typesafety matters
class UserFormModel {
    public firstName = '';
    public lastName = '';
    constructor(user?: Partial<UserFormModel>) {
        if (user) {
            Object.assign(this, { ...user });
        }
    }
}

@Component({
    ...
    template: `
    <form #form="ngForm" *ngIf="vm$|async as vm" (ngSubmit)="submit()">
        <label>
            First name
            <input type="text" [ngModel]="vm.user.firstName" name="firstName"/>
        </label>
        <label>
            Last name
            <input type="text" [ngModel]="vm.user.lastName" name="lastName"/>
        </label>
        <button>Submit form</button>
    </form>
    `,
})
// Extending from ObservableState makes this component a state machine
export class UserFormComponent extends ObservableState<{ user: UserForm }> implements AfterViewInit {
  // Get access to the reactive form, created by Angular behind the scenes
  @ViewChild('form') form!: NgForm; 
  // Create an observable ViewModel that uses the state
  public readonly vm$ = this.state$;

  constructor() {
    super();
    // Initialize our state with a new UserFormModel
    this.initialize({
      user: new UserFormModel(),
    });
  }

  public ngAfterViewInit(): void {
    // The reactive form is created by Angular and is ready in the
    // ngAfterViewInit lifecycle hook. Here we feed our local state with the changes
    // of our form
    this.connect({
      user: this.form.valueChanges?.pipe(
        map((v) =>  new UserFormModel({ ...this.snapshot.user, ...v }))
      ),
    });
  }

  public submit(): void {
    // This logs out the state of the form on submit
    console.log(this.snapshot.user);
  }
}
```

For a more in depth example, please read [this article](https://blog.simplified.courses/template-driven-or-reactive-forms-in-angular/){:target="_blank"} because we will use this approach for the rest of this article..
Like we mentioned before, using a `FormArray` in combination with template-driven forms is not possible. The `FormArray` class belongs to the reactive forms package and template-driven forms can only
create `FormGroup` and `FormControls` instances automatically by using the `ngModel` and `ngModelGroup` directives.

## The phonenumbers functionality

Let's continue with our phonenumbers example. We have a simple user form that has a `firstName` and `lastName` property, together with an array of phonenumbers.
FormArrays are not available in template-driven forms, but we could use the following technique to achieve something similar:

```html
<form #form="ngForm" *ngIf="vm$|async as vm" (ngSubmit)="submit()">
    <label>
        First name
        <input type="text" [ngModel]="vm.user.firstName" name="firstName"/>
    </label>
    <label>
        Last name
        <input type="text" [ngModel]="vm.user.lastName" name="lastName"/>
    </label>
    <div *ngFor="let phoneNumber of phoneNumbers; let i = index">
        <input type="text" [ngModel]="phoneNumbers[i]" name="phoneNumber{{i}}" />
    </div>
    ...
</form>
```

This would create a `FormControl` instance on the form for every phonenumber and **would pollute the automatically created reactive form** behind the scenes. It would look like this:

```typescript
ngForm = {
    form: {
        firstName: FormControl,
        lastName: FormControl,
        phonenumber0: FormControl,
        phonenumber1: FormControl,
        phonenumber2: FormControl,
        phonenumber3: FormControl,
    }
}
```
As we can see, an array is not possible. Angular would just create formControls with unique keys.
Let's stop trying to use arrays and use template-driven forms they way they were meant to be used. This means our form can only exist out of form groups and form controls.
This is the automatically created reactive form that we want to achieve:

```typescript
ngForm = {
    form: {
        firstName: FormControl,
        lastName: FormControl,
        // Clean separate formGroup that contains the phonenumbers
        phonenumbers: {
            0: FormControl,
            1: FormControl,
            2: FormControl,
            3: FormControl,
        }
    }
}
```

We can see that the form is not polluted anymore and we use a `phonenumbers` object with indexes as keys that will contain the `FormControl` instances for every phonenumber.

Let's update the `UserFormModel` since this is the model that will represent our actual form:

```typescript
class UserFormModel {
    public firstName = '';
    public lastName = '';
    // Will result to the input that will be used
    // to add a phone number
    public addPhonenumber = '';
    // The list of actual phone numbers
    public phonenumbers: { [key: string]: string } = {};
    ...
}
```

Ideally, an initial version of our form would look like this:

```html
<form ...>
    ...
    <h2>Phonenumbers</h2>
    <div ngModelGroup="phonenumbers">
        <div class="phonenumber" *ngFor="let key of vm.user.phonenumbers; trackBy: tracker">
            <input type="text" [ngModel]="vm.user.phonenumbers[key]" name="{%raw%}{{key}}{%endraw%}" />
        </div>
    </div>
    ...
</form>
```

**This will result in an error** because `vm.user.phonenumbers` **is not an iterable but an object** and the `*ngFor` directive expects an iterable like an Array.
To convert this object to an array we can use the `keyvalue` pipe. 

```html
<form #form="ngForm" *ngIf="vm$|async as vm" (ngSubmit)="submit()">
    ...
    <h2>Phonenumbers</h2>
    <div ngModelGroup="phonenumbers">
        <div class="phonenumber" *ngFor="let item of vm.user.phonenumbers|keyvalue; trackBy: tracker">
            <input type="text" [ngModel]="vm.user.phonenumbers[item.key]" name="{%raw%}{{item.key}}{%endraw%}" />
        </div>
    </div>
    ...
</form>
```

We can see that we use the key (index) in the `name` attribute.
We also use the `key` to bind the actual phonenumber to the `[ngModel]` directive. Note that we do not use the banana in the box syntax
because `this.form.valueChanges` is automatically feeding our component its state machine in the `ngAfterViewInit()` lifecycle hook.

## Completing the phonenumbers functionality with adding and deleting phonenumbers

The iterative part is ready but now we still need to be able to add and remove phonenumbers.

```html
<form #form="ngForm" *ngIf="vm$|async as vm" (ngSubmit)="submit()">
    ...
    <h2>Phonenumbers</h2>
    <div ngModelGroup="phonenumbers">
        <div class="phonenumber" *ngFor="let item of vm.user.phonenumbers|keyvalue; trackBy: tracker">
            <input type="text" [ngModel]="vm.user.phonenumbers[item.key]" name="{%raw%}{{item.key}}{%endraw%}" />
            <!-- Delete the phonenumber based on the key -->
            <button type="button" (click)="deletePhonenumber(item.key)">Delete phonenumber</button>
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
An array would be a little bit easier here, but since template-driven forms don't play nice with arrays we have to translate an array
to an object and the other way around.

Let's start with the `deletePhoneNumber()` method. We will use `Object.values` to get all the values from our phonenumbers object and create a new array
with the `addPhonenumber` value. This gives us a brand new array with the newly added phonenumber in it:

```typescript
public addPhonenumber(): void {
    // Create new array with all the old phonenumbers and new one
    const phonenumbers = [
        ...Object.values(this.snapshot.user.phonenumbers), 
        this.snapshot.user.addPhonenumber
    ];
}
```

What's left to do is patch our state with the newly calculated phonenumbers. For that we need to convert our
new array back to an object where all the keys are indexes. We can clean the `addPhonenumber` field at the same time.

```typescript
public addPhonenumber(): void {
    const phonenumbers = [ ... ];
    // Patch the state with the new phonenumbers
    // Angular will feed the form automatically for us
    this.patch({
        user: {
            // Take all the previous values
            ...this.snapshot.user, 
            // Create an object from our new array
            phonenumbers: arrayToObject(phonenumbers), 
            // Clear the add phonenumber field
            addPhonenumber: ''
        }
    })
}
```

To create an object from an array we can use the `reduce()` method that lives on the arrays prototype. We created a simple pure `arrayToObject` function
that can be reuced everywhere:

```typescript
// ['foo', 'bar', 'baz'] => {0: 'foo', 1: 'bar', 2: 'baz'}
function arrayToObject<T>(arr: T[]): { [key: number]: T } {
  return arr.reduce((acc, value, index) => ({...acc, [index]: value}), {})
}
```

For the `deletePhonenumber()` method we can use `Object.values` to get all the values from our phonenumbers object and use the `filter` method to make sure
that the phonenumber we want to delete isn't part of the array anymore.

```typescript
public deletePhonenumber(key: string): void {
    // Create new array with all phonenumbers, except the one we want deleted
    const phonenumbers = Object.values(this.snapshot.user.phonenumbers)
        .filter((v, index) => index !== Number(key))
    
}
```

Patching is done exactly the same as we did with the `addPhoneNumber()` method. Below you can see both methods implemented:

```typescript
public addPhonenumber(): void {
    // Calculate new phonenumbers
    const phonenumbers = [
      ...Object.values(this.snapshot.user.phonenumbers),
      this.snapshot.user.addPhonenumber,
    ];
    // Patch the state with an object created from the phonenumbers array
    this.patch({
        user: {
            ...this.snapshot.user,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        },
    });
}

public deletePhonenumber(key: string): void {
    // Calculate new phonenumbers
    const phonenumbers = Object.values(this.snapshot.user.phonenumbers).filter(
        (v, index) => index !== Number(key)
    );
    // Patch the state with an object created from the phonenumbers array
    this.patch({
        user: {
            ...this.snapshot.user,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        },
    });
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
class UserFormModel {
    public firstName = '';
    public lastName = '';
    public addPhonenumber = '';
    public phonenumbers: { [key: string]: string } = {};
    constructor(user?: Partial<UserFormModel>) {
        if (user) {
            Object.assign(this, { ...user });
        }
    }
}

@Component({
    selector: 'app-user-form',
    templateUrl: './user-form.component.html',
    styleUrls: ['./user-form.component.css'],
    standalone: true,
    imports: [CommonModule, FormsModule],
})
export class UserFormComponent
    extends ObservableState<{ user: UserFormModel }>
    implements AfterViewInit
{
    // Get access to form
    @ViewChild('form') form!: NgForm;
    
    // Expose viewModel
    public readonly vm$ = this.state$;
    
    tracker = (i: any) => i;

    constructor() {
        super();
        // initialize the state
        this.initialize({
            user: new UserFormModel(),
        });
    }

    // Connect the form to the state of the component
    public ngAfterViewInit(): void {
        this.connect({
            user: this.form.valueChanges?.pipe(
                map((v) => new UserFormModel({ ...this.snapshot.user, ...v }))
            ),
        });
    }

    public submit(): void {
        console.log(this.form);
    }

    public addPhonenumber(): void {
        const phonenumbers = [
            ...Object.values(this.snapshot.user.phonenumbers),
            this.snapshot.user.addPhonenumber,
        ];
        // Patch state with new phonenumbers
        this.patch({
            user: {
                ...this.snapshot.user,
                phonenumbers: arrayToObject(phonenumbers),
                addPhonenumber: '',
            },
        });
    }

    public deletePhonenumber(key: string): void {
        const phonenumbers = Object.values(this.snapshot.user.phonenumbers).filter(
            (v, index) => index !== Number(key)
        );
        // Patch state with new phonenumbers
        this.patch({
            user: {
                ...this.snapshot.user,
                phonenumbers: arrayToObject(phonenumbers),
                addPhonenumber: '',
            },
        });
    }
}
// Pure function that creates an object from an array
function arrayToObject<T>(arr: T[]): { [key: number]: T } {
    return arr.reduce((acc, value, index) => ({ ...acc, [index]: value }), {});
}

```

The complete result of the template looks like this. This template has no boilerplate, it is readable
and Angular takes care of everything for us:

```html
<form #form="ngForm" *ngIf="vm$ | async as vm" (ngSubmit)="submit()">
    <label>
        First name <input type="text" [ngModel]="vm.user.firstName" name="firstName" />
    </label>
    <label>
        Last name <input type="text" [ngModel]="vm.user.lastName" name="lastName" />
    </label>
    <h2>Phonenumbers</h2>
    <div ngModelGroup="phonenumbers">
        <div
            class="phonenumber" *ngFor="let item of vm.user.phonenumbers | keyvalue; trackBy: tracker">
            <input type="text" [ngModel]="vm.user.phonenumbers[item.key]" name="{%raw%}{{ item.key }}{%endraw%}" />
            <button type="button" (click)="deletePhonenumber(item.key)">
                Delete phonenumber
            </button>
        </div>
    </div>
    <input type="text" [ngModel]="vm.user.addPhonenumber" name="addPhonenumber" />
    <button type="button" (click)="addPhonenumber()">Add phonenumber</button>
    <button>Submit form</button>
</form>
```


You can play with the example on [Stackblitz here](https://stackblitz.com/edit/angular-1dfycz?file=src%2Fuser-form%2Fuser-form.component.ts){:target="_blank"}.

### Signals

We have to be ready for the future, that's why we can refactor this to signals as well.

```typescript
export class UserFormComponent implements AfterViewInit {
    // We still need access to the form
    @ViewChild('form') form!: NgForm;
    
    // Create a user signal that holds the form values
    public readonly user = signal(new UserFormModel());

    // Create a viewModel specifically for the template
    public readonly vm = computed(() => ({ user: this.user() }));
    
    tracker = (i: any) => i;

    public ngAfterViewInit(): void {
        // Update the user signal when the form changes
        this.form.valueChanges?.subscribe((v) => {
            this.user.update((old) => new UserFormModel({ ...old, ...v }));
        });
    }
    public submit(): void { ... }

    public addPhonenumber(): void {
        // Create new array based on the values in the user signal
        const phonenumbers = [
            ...Object.values(this.user().phonenumbers),
            this.user().addPhonenumber,
        ];
        // Update the user signal with the new array
        this.user.update((old) => ({
            ...old,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        }));
    }

    public deletePhonenumber(key: string): void {
        // Create new array based on the values in the user signal
        const phonenumbers = Object.values(this.user().phonenumbers).filter(
            (v, index) => index !== Number(key)
        );
        // Update the user signal with the new array
        this.user.update((old) => ({
            ...old,
            phonenumbers: arrayToObject(phonenumbers),
            addPhonenumber: '',
        }));
    }
}

```

You can play with the example on [Stackblitz here](https://stackblitz.com/edit/angular-ngwvfw?file=src%2Fuser-form%2Fuser-form.component.ts){:target="_blank"}.

Even more opinionated? If you want to see how we use the signal store from [this article](https://blog.simplified.courses/state-management-with-angular-signals/){:target="_blank"}, you can check the [stackblitz here](https://stackblitz.com/edit/angular-fjbs2a?file=src%2Fuser-form%2Fuser-form.component.ts){:target="_blank"} as wel.

### Conclusion

Template-driven forms are clean, have no boilerplate and we let Angular take care of all the hard work for us.
Using ObservableState is easy to create reactive forms out of template-driven forms so we maintain reactivity.
We can not use `FormArray` since it is a part of the reactive forms package in Angular. Since template-driven forms create `FormControl` and `FormGroup` instances
for us automatically we can create clean form structures by converting arrays to objects and the other way around.
Refactoring this to signals was as easy as replacing `*ngIf="vm$ | async as vm"` with `*ngIf="vm() as vm"` and our class got cleaned up nice as well.
