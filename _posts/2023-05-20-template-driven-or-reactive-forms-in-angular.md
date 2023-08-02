---
layout: post
title:  "Template-driven or reactive forms in Angular"
date:   2023-05-23
published: true
comments: true
categories: [Angular, Forms, State management, ObservableState]
cover: assets/template-driven-or-reactive-forms-in-angular.jpg
description: "Should we use template-driven forms or reactive forms in Angular"
---

## Introduction

When Angular was released in 2016, the only solution they had for creating forms was template-driven forms. A principle where directives in the template were used to create forms. In Angular 4 the core team introduced a new concept called **reactive forms**.
It was a new way with a reactive API that exposed RxJS Observables when we wanted them. Pretty soon everyone was agreeing that reactive forms were the new way to go. The new best practice. Template-driven forms were even frowned upon and I have been advising companies to use Reactive forms for years. In Reactive forms, we would use `FormBuilder` to create `FormGroup` instances with `FormControl` and `FormArray` instances.
An argument that keeps coming back supporting Reactive Forms is that **they were better testable**. We are not sure how valid that point is since we see it as a best practice to test our components including our templates.
Another argument was **type-safety**. Even though reactive forms weren't type-safe for a lot of years. They are now, and template-driven forms are not.
Here are 3 other reasons why some developers favor reactive forms over template-driven forms:
- Manual event binding on inputs can introduce a lot of work. They rather subscribe to the `valueChanges` observable of a `FormControl` or `FormGroup` if they want to listen to those kinds of changes.
- Validations are shattered across the entire template making it a lot of work and less dynamic.
- They can leverage RxJS a lot more by combining observables, using `debounceTime()`, `switchMap()`, etc...

When watching 2 talks from [Ward Bell](https://twitter.com/wardbell){:target="_blank"} at ngConf and reading articles from [Tim De Schryver](https://timdeschryver.dev/blog/a-practical-guide-to-angular-template-driven-forms){:target="_blank"} and talking with [Jeffrey Bosch](https://twitter.com/jefiozie){:target="_blank"}, I started to realize that I had to take a closer look at template-driven forms and re-evaluate how I deal with forms today.

After doing tons and tons of research, and deciding to update some major applications to template-driven forms for a big client of mine, I have to admit that switching back to template-driven forms made forms fun again.

The goal of this article is to show why template-driven forms can make your life as an Angular developer easier and how easy it is to make them **reactive after all**.
This article will have more follow-up articles where we will tackle specific implementations in depth: Like validations etc.
We will focus on how to **make template-driven forms reactive** and how we can enjoy the best parts of both worlds.

## Let Angular work for you

Did you know that template-driven forms create `FormGroup` instances and `FormControl` instances for us behind the scenes?
Well they do, and Ward Bell calls it: **"Let angular do the work for you"**. The truth is: when writing template-driven forms we have to write a lot less code. We don't have to worry about manually creating our `FormGroup` and `FormControl` instances. Angular does that automatically for us.

Take this reactive forms component for instance:

```typescript
@Component({
  selector: 'my-app',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
  <form [formGroup]="form" (ngSubmit)="submit(form.value)">
      <label>
        First name
        <input type="text" [formControl]="form.controls.firstName"/>
      </label>
      <label>
        Last name
        <input type="text" [formControl]="form.controls.lastName"/>
      </label>
      <h2>Password</h2>
      <label>
        Password
        <input
          type="password"
          [formControl]="form.controls.passwords.controls.password"/>
      </label>
      <label>
        Confirm password
        <input
            [formControl]="form.controls.passwords.controls.confirmPassword"
            type="password"
        />
      </label>
      <button>Submit form</button>
  </form>
  `,
})
export class App {
  private readonly formBuilder = inject(FormBuilder);
  public readonly form = this.formBuilder.group({
    firstName: [''],
    lastName: [''],
    passwords:  this.formBuilder.group({
      password: [''],
      confirmPassword: [''],
    }),
  });

  public submit(): void {}
}
```

We have a fairly clean template, and we had to create some sort of composition with the `FormBuilder` to create the Reactive form. We can add validations as the second value in the arrays and we can extract observables from our form.
Now look at the template-driven variant of this form:

```typescript
class User {
  public firstName = '';
  public lastName = '';
  public passwords = {
    password: '',
    confirmPassword: ''
  };
}
@Component({
  selector: 'my-app',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
  <form #form="ngForm" (ngSubmit)="submit(form.value)">
      <label>
        First name
        <input type="text" [(ngModel)]="model.firstName" name="firstName"/>
      </label>
      <label>
        Last name
        <input type="text" [(ngModel)]="model.lastName" name="lastName"/>
      </label>
      <h2>Password</h2>
      <div ngModelGroup="passwords">
        <label>
          Password
          <input
            type="password"
            [(ngModel)]="model.passwords.password" name="password"/>
        </label>
        <label>
          Confirm password
          <input
            [(ngModel)]="model.passwords.confirmPassword" name="confirmPasswords"
            type="password"
          />
        </label>
      </div>
      <button>Submit form</button>
  </form>
  `,
})
export class App {
  public model = new User();
  public submit(value: any): void {
    console.log(value);
  }
}
```

We have created a clean `User` class that is completely type-safe and even has initial values. The typescript part of `App` has been simplified drastically. 
In the form element, we see that we have added the `#form="ngForm"` syntax to tell Angular this is our template-driven form.
For the rest, we use a combination of `ngModel`, `name` and `ngModelGroup` to create our form.
The syntax of this form is quite simple and because we use the correct values in the `name` and `ngModelGroup` Angular has created the perfect reactive form for us behind the scenes, automatically!

## Getting access to the reactive form

Template-driven forms do all the work for us, but what good are they if we can't access the reactive form. We want to be able to get observables from it right? It turns out this is quite easy:

```typescript
@Component({
  ...
  template: `
  <form #form="ngForm" (ngSubmit)="submit()">
      ...
  </form>
  `,
})
export class App {
  @ViewChild('form') form!: NgForm;
  ...

  public submit(): void {
    console.log(this.form);
  }
}
```

We are using the `@ViewChild()` decorator to get access to the form, and we log it to the console in the `submit()` method.
Let's check out what this logs:

![Alt text](../assets/template-driven-or-reactive-forms-in-angular/template-driven-reactive-forms.png)

This looks exactly like what `FormBuilder` would have made for us, but Angular did all the work for us!

## But what about validations?

Validations are a pain, and we definitely don't want to create custom validators and directives for them and add them everywhere in our templates. That is true, but I also don't like to see them in our reactive forms.
In [this talk](https://www.youtube.com/watch?v=EMUAtQlh9Ko){:target="_blank"} from Ward Bell, Ward explains that we shouldn't think about validation as form-validations but rather model-validations. We want to reuse these validations in different places and we want to validate a model in one place.
We don't want to go looking through our template-driven or reactive forms where all these validations lie.

Removing validations from the forms and putting them in models brings us some advantages:
- They are easy to find in one place
- They are reusable on other places of the app and even on a node.js backend
- They are composable and functional
- They are very easy to test
- They avoid boilerplate
- We can just glue them to our `FormControl`'s and `FormGroup`'s automatically

Since we want to keep this article to the point, we will tackle validations and remove validation-message-boilerplate in a next article. If you can't wait, drop me a DM and I will share some stackblitz code with you.
So for now, let's don't talk about validations anymore, but we promise we will tackle it in one of our next articles in depth.

## Making it reactive

Making it reactive, however, is an important part of this article. What we want but currently don't have is:
- Type safety on `form`
- Getting observables from our form
- Getting observables from groups or controls of our form
- We don't want to have to bind all the events manually in the template. We want to work against one reactive form (which was the whole point of Reactive forms)

I have created a small class (<100 lines of code) that is called [ObservableState](https://github.com/simplifiedcourses/observable-state/){:target="_blank"} that I explain in depth in [this article](https://blog.simplified.courses/observable-state-in-angular/){:target="_blank"} and it turns out this can make your form reactive in a few lines of code.
The whole point was having an opinionated way of doing state management that is super light weight and **simplifies state management in angular completely!**.

The first thing we need to do is adapt the `UserClass` so it can take partial updates:

```typescript
class User {
  public firstName = '';
  public lastName = '';
  public passwords = {
    password: '',
    confirmPassword: '',
  };
  constructor(user?: Partial<User>) {
    if (user) {
      Object.assign(this, { ...user });
    }
  }
}
```

Now we need to make the `App` extend from `ObservableState<T>`, initialize it with default values (`initialize()`),
and connect the form after the view is initialized on the `ngAfterViewInit()` lifecycle hook:

```typescript
@Component({
  ...
  template: `
  <form #form="ngForm" (ngSubmit)="submit()">
  ...
  `,
})
export class App extends ObservableState<{ user: User }> implements AfterViewInit {
  @ViewChild('form') form!: NgForm;
  ...

  constructor() {
    super();
    // initialize the state
    this.initialize({
      user: new User(),
    });
  }

  // Wait until Angular has created the initial version of the form
  public ngAfterViewInit(): void {
    // connect it with the form
    this.connect({
      user: this.form.valueChanges?.pipe(
        map((v) => new User({ ...this.snapshot.form, ...v }))
      ),
    });
  }
  public submit(): void {
    // The state becomes the single source of truth
    console.log(this.snapshot.user);
  }
}

```

Now we can create a [ViewModel](https://blog.simplified.courses/reactive-viewmodels-for-ui-components-in-angular/){:target="_blank"} for that with the `onlySelectWhen()` method from `ObservableState`, and we can even drop the banana in the box syntax and work with one-way-databinding since the `valueChanges` of the (by angular) generated `form` feeds `ObservableState`.

```typescript
@Component({
  ...
  template: `
  <form #form="ngForm" *ngIf="vm$|async as vm" (ngSubmit)="submit()">
      <label>
        First name
        <input type="text" [ngModel]="vm.user.firstName" name="firstName"/>
      </label>
      ...
      <button>Submit form</button>
  </form>
  `,
})
export class App extends ObservableState<{ user: User }> implements AfterViewInit {
  ...
  public readonly vm$ = this.onlySelectWhen(['user']);
  ...
}
```

### But what about all the other goodies reactive forms have to offer us?

You are talking about the status, right?! Whether controls are `valid`, `dirty`, etc...
If you care about these states, then we can just add them to the state:

```typescript
@Component({...})
// Update the type so it also supports dirty and valid
export class App extends ObservableState<{ user: User, dirty: boolean, valid: boolean }> implements AfterViewInit {
  ...
  constructor() {
    super();
    this.initialize({
      ...
      // initialize the new state values
      // will throw an error as form is not defined when constructor is called
      dirty: this.form.dirty,
      valid: this.form.valid
    })
  }

   public ngAfterViewInit(): void {
    this.connect({
      ...
      // Connect both states to the state
      dirty: this.form.statusChanges.pipe(map(() => this.form.dirty)),
      valid: this.form.statusChanges.pipe(map(() => this.form.valid))
    });
  }
}
```

Now every time the form becomes dirty, it will update the state which can give us an `Observable`, but also a snapshot.
The only thing left to do is translate to a ViewModel and we can use it in a reactive way in our template:

```typescript
@Component({
  ...
  template: `
  <form #form="ngForm" *ngIf="vm$|async as vm" (ngSubmit)="submit()">
      Dirty: {{vm.dirty}}
      Valid: {{vm.valid}}
      ...
      <button>Submit form</button>
  </form>
  `,
})
// Update the type so it also supports dirty and valid
export class App extends ObservableState<{ user: User, dirty: boolean, valid: boolean }> implements AfterViewInit {
  ...
  public readonly vm$ = this.onlySelect(['user', 'dirty', 'valid']);
  ...
}
```

### Enabling and disabling controls

Do you remember how annoying it was to enable and disable controls based on how other controls changed in Reactive forms?
**This becomes complex really fast!**:

```typescript
ngOnInit(): void {
  this.form.controls.passwords.password.valueChanges
    .pipe(
      takeUntil(this.destroy$$)
    )
    .subscribe(value => {
      if(value === ''){
        this.form.controls.passwords.confirmPassword.disable():
      } else{
        this.form.controls.passwords.confirmPassword.disable():
      }
    })
}

```
This isn't declarative at all and the bigger our component gets, the harder it would become to maintain.
It's nice to use the declarative approach and just calculate it in the ViewModel.

Now you can do most of that logic in a declarative way in the ViewModel. Look how clean this got!

```typescript
@Component({
  ...
  template: `
  <form #form="ngForm" *ngIf="vm$|async as vm" (ngSubmit)="submit()">
      ...
      <input [disabled]="vm.confirmPasswordDisabled" .../>
       ...
  </form>
  `,
})
export class App extends ObservableState<{ user: User }> implements AfterViewInit {
  ...
  public readonly vm$ = this.onlySelectWhen(['user']).pipe(
    map(state => {
      return {
        user: state.user,
        confirmPasswordDisabled: state.user.passwords.password === ''
      }
    })
  );
  ...
}
```

We could take it even further and remove the passwords when there is no first name filled in. We could use an `*ngIf` directive for that and YES! Angular will automatically remove and recreate a `FormGroup` with 2 `FormControls` for us. The only thing that we need to do is add a `showPasswords` property in our ViewModel and calculate it there:

```typescript
@Component({
  ...
  template: `
  <form #form="ngForm" *ngIf="vm$|async as vm" (ngSubmit)="submit()">
      ...
      <div ngModelGroup="passwords" *ngIf="vm.showPasswords">
        ...
      </div>
      ...
  </form>
  `,
})
export class App extends ObservableState<{ user: User }> implements AfterViewInit {
  ...
  public readonly vm$ = this.onlySelectWhen(['user']).pipe(
    map(state => {
      return {
        ...
        showPasswords: state.user.firstName !== ''
      }
    })
  );
  ...
}
```

Try doing that with a reactive form and keep it simple at the same time. By using this approach we let Angular work for us and we still benefit from completely type-safe reactivity, but without all the boilerplate code and complexity...

## Specific reactivity

Now we can only create Observables from the entire form, but in some cases, we want to fetch new data, add form controls, remove form controls whenever things change. Since our form is connected to `ObservableState` this is easy to do:
We just have to use the `onlySelectWhen()` method and connect that to the state. `ObservableState` will automatically perform a `distinctUntilChanges()` behind the scenes so the observable will only emit when needed:


```typescript
@Component({...})
export class App extends ObservableState<{ user: User, firstName: string }> implements AfterViewInit {
  ...

  constructor() {
    super();
    this.initialize({
      user: new User(),
      firstName: '' // initialize
    });
    // get notified when firstName changes
    this.select('firstName').subscribe(() => {
      console.log('first name has changed')
    })
  }

  public ngAfterViewInit(): void {
    this.connect({
      ...
      // Just connect it to the state
      firstName: this.onlySelectWhen(['user']).pipe(
        map(state => state.user.firstName)
      )
    });
  }
  ...
}
```

This can turn into a very declarative approach. In the next example, we see how we can load zip codes when the country of the form changes. Everything is automatically added to the state and we can get `Observable`'s and snapshots from it.

```typescript
 this.connect({
      country: this.onlySelectWhen(['user']).pipe(
        map(state => state.user.country)
      ),
      zipcodes: this.onlySelectWhen(['country']).pipe(
        switchMap(country => this.countryService.fetchZipcodesByCountry(country))
      )
    });
```

## Summary

Reactive forms have benefits, and so have template-driven forms. But template-driven forms create reactive forms behind the scenes.
Angular does that automatically for us. Using template-driven forms with `ObservableState` will give your form steroids and will get the best from both worlds:
- Remove boilerplate
- Clean templates: no `(change)` expressions shattered in the template.
- Turn forms into easy-to-read declarative code
- Give you observables but also snapshots. It will be easy to get signals from them as well.
- Create a form based on a simple class with initial values

Also, check out these 2 talks from ward bell:
- [Prefer Template-Driven Forms](https://www.youtube.com/watch?v=L7rGogdfe2Q&t=3s){:target="_blank"}
- [Form Validation Done Right ](https://www.youtube.com/watch?v=EMUAtQlh9Ko){:target="_blank"}

Interested in playing with the code? Check out [this stackblitz](https://stackblitz.com/edit/angular-wqxqyb){:target="_blank"}!

Thanks to the awesome reviewers;
- [Daniel Glejzner](https://twitter.com/danielglejzner){:target="_blank"}
- [Jeffrey Bosch](https://twitter.com/jefiozie){:target="_blank"}

If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training){:target="_blank"} and [Angular Coaching](https://www.simplified.courses/angular-coaching){:target="_blank"}
