---
layout: post
title:  "Reorder conditional table columns in Angular"
date:   2023-06-29
published: true
comments: true
cover: assets/reorder-conditional-table-columns-in-angular.jpg
categories: [Angular]
description: "In this article, we will see how we can use content projection to create reorderable conditional table columns"
---

At one of our clients, we had to implement a complex table in Angular. Some features of this table were:
- Expandable rows
- Resizable columns width drag and drop
- Select/multi-select with control and shift keys
- Multisorting
- Pagination

The reason why we wrote this ourselves is that this component is core to the business of our client and they want very specific logic.
A new feature that was requested was the ability to **conditionally show columns** based on keys.
We would have a string array that contained keys that we would pass to the table component. The next feature request was to determine the order of the columns based on that array:

```typescript
@Component({
    template: `
    <table [showColumns]="columnsToShow">
        ...
    </table>
    `
})
export class AppComponent{
columnsToShow = ['firstName', 'lastName', 'age']

}
```

In this pseudo code, we see that a table would have to render the `firstName`, `lastName` and `age` columns in that order.
In this article, we will create a simple table that will render columns conditionally and in the right order.

### First simple solution

The first solution is to use `*ngFor` and render the columns automatically:

```typescript
@Component({
    ...
    template: `
    <table>
        <thead>
        <tr *ngFor="let column of columnsToShow">
            <th>{% raw %}{{column}}{% endraw %}</th>
        </tr>
        </thead>
        <tbody>
            <tr *ngFor="let user of users">
                <td *ngFor="let column of columnsToShow">
                    {% raw %}{{user[column]}}{% endraw %}
                </td>
            </tr>
        </tbody>
    </table>
  `,
})
export class App {
  public readonly columnsToShow = ['firstName', 'lastName', 'age', 'gender'];
  public readonly users = [
    {
       firstName: 'Brecht',
       lastName: 'Billiet',
       age: 35,
       gender: 'male'
    }
    ...
  ];
}
```

This approach is very limited. It does not support specific implementations in the `th` or `td` elements.
For instance, showing an icon for the gender would be very hard, since every `td` template would be the same.

### A second approach (idea)

The second idea that we had was to use structural directives like this:

```html
    <table [columnsToShow]="columnsToShow">
        <thead>
        <tr *ngFor="let column of columnsToShow">
            <th>{% raw %}{{column}}{% endraw %}</th>
        </tr>
        </thead>
        <tbody>
            <tr *ngFor="let user of users">
                <td *conditionalTd="firstName">
                    {% raw %}{{user.firstName}}{% endraw %}
                </td>
                <td *conditionalTd="lastName">
                    {% raw %}{{user.lastName}}{% endraw %}
                </td>
                <td *conditionalTd="age">
                   {% raw %}{{user.age}}{% endraw %}
                </td>
                <td *conditionalTd="gender">
                   {% raw %}{{user.gender}}{% endraw %}
                </td>
            </tr>
        </tbody>
    </table>
```

While this approach would work for conditionally showing columns, it would not work for the reordering of columns.
`firstName` would always be shown first, and `lastName`, `age` and `gender` would always be the next columns that would get rendered.
Since inheritance between child structural directives is not possible in Angular, we had to find a new and better solution.

### Fixing the issue with content projection

It started to make sense that since the `td` and `th` elements had to be shown conditionally and the order was of importance, we had to use `ng-template` elements.
We would create an `[conditional-templates-with-order]` attribute component that would be used on the `tr` element. In that component, we would use content-projection to project 4 `ng-template` elements:

```html
<tr [conditional-templates-with-order]="columnsToShow">
    <ng-template>
        <td>{% raw %}{{user.firstName}}{% endraw %}</td>
    </ng-template>
    <ng-template>
        <td>{% raw %}{{user.lastName}}{% endraw %}</td>
    </ng-template>
    <ng-template>
        <td>{% raw %}{{user.age}}{% endraw %}</td>
    </ng-template>
    <ng-template>
        <td>{% raw %}{{user.gender}}{% endraw %}</td>
    </ng-template>
</tr>
```

Since we have to identify the `ng-template` elements separately, we created a `conditional-template` directive where we can pass a key to as an `@Input()` property:

```typescript
@Directive({
    selector: '[conditional-template]',
    standalone: true
})
export class ConditionalTemplateDirective {
    @Input('conditional-template') public conditionalTemplate = '';
}
```

We could use the new directive like this:
```html
<tr [conditional-templates-with-order]="columnsToShow">
    <ng-template conditional-template="firstName">
        <td>{% raw %}{{user.firstName}}{% endraw %}</td>
    </ng-template>
    <ng-template conditional-template="lastName">
        <td>{% raw %}{{user.lastName}}{% endraw %}</td>
    </ng-template>
    <ng-template conditional-template="age">
        <td>{% raw %}{{user.age}}{% endraw %}</td>
    </ng-template>
    <ng-template conditional-template="gender">
        <td>{% raw %}{{user.gender}}{% endraw %}</td>
    </ng-template>
</tr>
```
This seems like a solid structure to build our tables. Before we continue to the implementation of `conditional-templates-with-order` let's show the entire component with structure.

```html
<table>
    <thead>
        <tr [conditional-templates-with-order]="columnsToShow">
            <!-- We could do an *ngFor but this is more flexible -->
            <ng-template conditional-template="firstName">
                <td>First name</td>
            </ng-template>
            <ng-template conditional-template="lastName">
                <td>last name</td>
            </ng-template>
            <ng-template conditional-template="age">
                <td>Age</td>
            </ng-template>
            <ng-template conditional-template="gender">
                <td>Gender</td>
            </ng-template>
        </tr>
    </thead>
    <tbody>
        <!-- Simply loop over the users -->
        <tr 
            *ngFor="let user of users; trackBy: tracker"
            [conditional-templates-with-order]="columnsToShow">
            <ng-template conditional-template="firstName">
                <td>{% raw %}{{user.firstName}}{% endraw %}</td>
            </ng-template>
            <ng-template conditional-template="lastName">
                <td>{% raw %}{{user.lastName}}{% endraw %}</td>
            </ng-template>
            <ng-template conditional-template="age">
                <td>{% raw %}{{user.age}}{% endraw %}</td>
            </ng-template>
            <ng-template conditional-template="gender">
                <td>{% raw %}{{user.gender}}{% endraw %}</td>
            </ng-template>
        </tr>
    </tbody>
</table>
```

```typescript
@Component({
    ...
})
export class App {
    // All the available columns
    public readonly columnsToShow = ['firstName', 'lastName', 'age', 'gender'];

    // Our list of users
    public readonly users = [
        {
            firstName: 'Brecht',
            lastName: 'Billiet',
            age: 35,
            gender: 'male',
        },
        {
            firstName: 'Silvie',
            lastName: 'Rayée',
            age: 32,
            gender: 'female',
        },
    ];
}
```

### Implementing ConditionalTemplatesWithOrderComponent

The last thing we need to do is implement the `ConditionalTemplatesWithOrderComponent`.
Since we are using an attribute this might look like a directive, but it is not.
It's a component that we can use on any type of element. In our case a `tr` element.
The responsibility of this component is:
- Looping over the passed columns to show
- Creating a **template outlet** for all these columns
- Look for all the `ng-template` children and `ConditionalTemplateDirective` children
- Look up the right index of the template
- Pass the right template to the component

To make the selector a little bit more strict, let's make sure that we can only use the component on a `tr` element. Since we want to pass columns to show directly to the template let's create the input like this:

```typescript
@Component({
    ...
    // selector can only be used on a tr element
    selector: 'tr[conditional-templates-with-order]',
})
export class ConditionalTemplatesWithOrderComponent {
    // Avoid boilerplate
    @Input('conditional-templates-with-order') protected order: string[] = [];

    protected getTemplate(key: string): TemplateRef<any>|null {
        throw new Error('not implemented yet');
    }
}
```

In our template, we would loop over all the keys of our columns and create a new `*ngTemplateOutlet` for every key and we will pass it  a `TemplateRef` that we want to receive through a `getTemplate(key)` method:

```html
<ng-container *ngFor="let key of order; trackBy: tracker">
    <ng-container *ngTemplateOutlet="getTemplate(key)"></ng-container>
</ng-container>
```

#### Getting access to the children

To render the right `TemplateRef` at the right place we need access to all the `TemplateRef` children and all the  `ConditionalTemplateDirective` children of this component. Remember how it is being used (our `ConditionalTemplateDirective` always has an `ng-template` with a `conditional-template` directive applied to it):

```html
```html
<tr [conditional-templates-with-order]="columnsToShow">
    <ng-template conditional-template="firstName">
       ...
    </ng-template>
   ...
</tr>
```

To get access to those children, we can use `ContentChildren`:

```typescript
export class ConditionalTemplatesWithOrderComponent {
    // Get access to `ng-template`
    @ContentChildren(TemplateRef)
    protected templates!: QueryList<TemplateRef<any>>;
    
    // Get access to `conditional-template`
    // which holds the key
    @ContentChildren(ConditionalTemplateDirective) 
    protected conditionalTemplateDirectives!: QueryList<ConditionalTemplateDirective>;

    @Input('conditional-templates-with-order') protected order: string[] = [];

    ...
}
```

#### Implementing the getTemplate() method

The only thing we need to do is to implement the `getTemplate()` method which will loop over the `ConditionalTemplateDirective` instances, map it to the actual key that we passed and locate the index of the key passed with the `getTemplate()` method.
When we have the index, we can just use the list of `ng-template` references and return the right `TemplateRef` instance:

```typescript
export class ConditionalTemplatesWithOrderComponent {
    ...

    protected getTemplate(key: string): TemplateRef<any> | null {
        // Get the index in the templates based on the key
        const index = this.conditionalTemplateDirectives
            .toArray()
            .map((item) => item.conditionalTemplate)
            .indexOf(key);
            
        // return the right template
        return this.templates.toArray()[index];
    }
}
```
Here is the entire implementation of the `ConditionalTemplatesWithOrderComponent`: 

```typescript
@Component({
  selector: 'tr[conditional-templates-with-order]',
  standalone: true,
  imports: [CommonModule],
  template: `
  <ng-container *ngFor="let key of order; trackBy: tracker">
    <ng-container *ngTemplateOutlet="getTemplate(key)"></ng-container>
  </ng-container>
  `,
})
export class ConditionalTemplatesWithOrderComponent {
    // Get access to `ng-template`
    @ContentChildren(TemplateRef)
    protected templates!: QueryList<TemplateRef<any>>;
    
    // Get access to `conditional-template`
    // which holds the key
    @ContentChildren(ConditionalTemplateDirective) 
    protected conditionalTemplateDirectives!: QueryList<ConditionalTemplateDirective>;


    @Input('conditional-templates-with-order') protected order: string[] = [];

    // TrackBy for performance
    protected tracker = (i: number) => i;

    protected getTemplate(key: string): TemplateRef<any> | null {
        // Get the index in the templates based on the key
        const index = this.conditionalTemplateDirectives
        .toArray()
        .map((item) => item.conditionalTemplate)
        .indexOf(key);

        // return the right template
        return this.templates.toArray()[index];
    }
}
```

### Test the logic with a multiselect and reverse button

The implementation is finished, but we still want to test if everything works.
Let's add 2 things to our application:
- A multi-select where we can select the columns that need to be shown
- A button to reverse the order

Since [template-driven forms](https://blog.simplified.courses/template-driven-or-reactive-forms-in-angular/){:target="_blank"}
 are awesome let's create a simple `select` to determine the columns that need to be shown and let's add a button to reverse the order of our columns:

```html
<select
    [(ngModel)]="columnsToShow"
    name="columnsToShow" multiple>
    <option [ngValue]="col" *ngFor="let col of allColumns">
        {% raw %}{{ col }}{% endraw %}
    </option>
</select>
<button (click)="reverse()">Reverse the order</button>
```

```typescript
export class App {
    ...
    public readonly allColumns = ['firstName', 'lastName', 'age', 'gender'];
    public columnsToShow = [...this.allColumns];

    public reverse(): void {
        this.columnsToShow = [...this.columnsToShow.reverse()];
    }
}
```

Check out the entire solution in StackBlitz [here](https://stackblitz.com/edit/stackblitz-starters-eeqpda?file=src%2Fmain.ts){:target="_blank"}

<iframe src="https://stackblitz-starters-eeqpda.stackblitz.io/" width="100%" height="500" style="border:none"></iframe>  

### Summary

Using content projection in Angular can be something very powerful. In this article, we learned how to use `ng-template` in combination with `@ContentChildren()` and `*ngTemplateOutlet` to not only conditionally render elements, but also determine their order. I hope you enjoyed the article!

If you like to learn directly from me, check out my [Angular Training](https://www.simplified.courses/angular-training){:target="_blank"} and [Angular Coaching](https://www.simplified.courses/angular-coaching){:target="_blank"}
