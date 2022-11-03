---
layout: post
title:  "Smart components, dumb components and facades in Angular"
date:   2022-11-03
published: false
categories: Angular architecture
---
This article is about 2 concepts that we can use in Angular applications.
Since these are architectural principles, they could be applied on other technologies as well.
Whether you are an Angular developer, a React developer or a developer that works in any other type of component based javascript framework, these
principles should apply for your application as well. The principles explained in this article are called:
- Smart and Dumb components
- Facades

These principles are explained in this article because they have one thing in common: We use them to keep components ignorent.

## Smart & Dumb components

For the principle of Smart & Dumb components, the goal is to seperate our components into 2 groups of components based on their responsibilities.

In one group we have the dumb components which are also called presentational components or ui components.
In the other group we have the smart components which are also called container components or orchestration components.

### Now what is a dumb component?

A dumb component is stupid in the way that it is ignorent from the rest of the application. It doesn't know about the data services that are fetching the data from our apis, nor does it now about how the application handles state management. Whether data is real-time or not, whether the data comes in through observables our plain javascript objects, the component doesn't know and has only 2 very clear responsibilities:

- It should render its DOM correctly based on `@Input` properties it will get from his parent component
- It should notify their parents with new possible data, but it should not care what happens with that data.

This is an example of a simple dumb component, that should render the name in the DOM and should notify its parent when the name has changed:

```typescript
@Component({
selector: 'hello',
template: `
    <h1>Hello {{name}}!</h1>
    <input type="text" [(ngModel)]="name" 
        (input)="nameChange.emit(name)">
`,
})
export class HelloComponent  {
    @Input() name: string;
    @Output() nameChange = new EventEmitter<string>()
}

```

Notice that these components usually don't have any dependencies injected into them.

There are some other things that a dumb component should never do:
- It should never communicate with the rest of the application.
- It should never mutate state, it's not its responsibility.
- It should never call to external state that is not passed by `@Input` properties. However it can manage its own state.

A dumb component **can** however contain complex logic. It does not mean that because we call them dumb components that they can't contain complexities. For instance a calendar's month view would have to calculate leap years, starting the week from monday instead of sunday and would have calculate an amount of week rows to show dependening on the month.

Even though that complexities can be part of those component, this month view should never fetch data itself. That would be the responisiblity of the smart components.

What are the benefits of dumb components?
- They are easy to test
- They are easy to reuse
- It follows the separation of concerns principle
- It makes it easier to follow the dataflow and find bugs in the application

### What is a smart component?

A smart component will orchestrate. It would initiate XHR calls and state management. When data is being fetched it would pass that data to its dumb components that would have their own responsibility in presenting that data.
Smart components would also trigger navigations Eg: when a POST call succeeds.

Smart components would have dependencies injected into them because they need those dependencies to orchestrate:
A simple example can be found below: We pass the name from the smart component to the dumb `hello` component that will notify the smart component when it's changed, and when we click on the save button the smart component will use the `nameService` to save the name, and navigate away on success:

```typescript
@Component({
selector: 'name',
template: `
    <hello name="{{ name }}" (nameChange)="name=$event"></hello>
    <button (click)="save()">Save</button>
`
})
export class NameComponent  {
  private readonly nameService = inject(NameService);
  name = 'Brecht';

  public save(): void {
    this.nameService.save(this.name).then(() =>{
      this.router.navigate(['..'])
    })
  }
}

```
