---
layout: post
title:  "Smart components, ui components and Sandbox facades in Angular"
date:   2022-11-14
published: true
comments: true
categories: Angular architecture
cover: assets/smart-components-ui-components-sandbox-facades.jpg

---
This article is about 2 concepts that we could use in our Angular applications.
Since these are architectural principles, they could be applied to other technologies as well.
Whether you are an Angular developer, a React developer or a developer that works in any other
type of component-based javascript framework, these
principles could apply to your application as well. The principles explained in this article are called:
- Smart and Ui components
- Sandbox facades

These principles are explained in this article because they have one thing in common: We use them to enforce the separation of concerns and the single responsibility principle.

## Smart & ui components

For the principle of Smart & ui-components, the goal is to separate our components into 2 groups of components based on their responsibilities.

In one group we have the ui components (also called dumb components in the past, but that could be conceived as offensive) which are also called presentational components.
In the other, group we have the smart components which are also called container components.

### What is a ui component?

A ui component is dumb in the way that it doesn't know from the rest of the application. 
It doesn't know about the data services that are fetching the data from our API's, nor does it know about how the application handles global state management. 
Whether data is real-time or not, whether the data comes in through observables with async pipes or plain javascript objects, the component doesn't know and has only 2 very clear responsibilities:

- It should render its DOM correctly based on data flowing through the `@Input` properties that will be provided by its parent component.
- It should notify their parent of new events, but it should not care what happens with those events.

This is an example of a simple ui component, that renders the name in the DOM and should notify its parent when the name has changed:

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

There are some other things that a ui component should never do:
- It should never communicate with the rest of the application.
- It should never mutate data, that's not its responsibility.
- It should never call to external state that is not passed by `@Input` properties. However, it can manage its own state.

A ui component **can** however contain complex logic. It does not mean that because they are called dumb sometimes, that they can't contain complexity. 
For instance, a calendar's month view would have to calculate leap years, starting the week from Monday instead of Sunday and would have to calculate an amount of week rows to show depending on the month.

Even though this kind of component can contain such complexity, the *month view* should never fetch data itself. That would be the responsibility of the smart components.

What are the benefits of a components?
- They are easy to test
- They are easy to reuse
- It follows the separation of concerns principle
- It makes it easier to follow the dataflow and find bugs in the application
- We can optimize our application for performance regarding Change Detection

In the tweet below you see that we have created an ebook about [Change Detection](https://www.simplified.courses/angular-change-detection-simplified-e-book){:target="_blank"}.

### What is a smart component?

A smart component will orchestrate. It would initiate XHR calls and state management. When data is being fetched it would pass that data to its dumb components that would have their own responsibility in presenting that data.
Smart components would also trigger route changes, for example when a POST call succeeds.

Smart components would have dependencies injected into them because they need those dependencies to orchestrate:
A simple example can be found below: We pass the name from the smart component to the ui `hello` component that will notify the smart component when it changed, and when we click on the save button the smart component will use the `nameService` to save the name and navigate away on success:

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

### Folder structure

Even though this is obviously highly opinionated, we came up with a folder structure that looks like this:

- `feat-lib/lib/src/components/ui/foo/foo.ui-component.ts`
- `feat-lib/lib/src/components/smart/bar/bar.smart-component.ts`

When we use Nrwl devtools, we could leverage the following command to create this:

```shell
npx nx g @nrwl/angular:component components/ui/foo --type=ui-component --project=feat-lib
```

## Sandbox facades

Smart components usually live in feature libraries. Feature libraries are Angular modules or libraries 
(usually behind a barrel file) that contain presentation logic and not just pure ui logic. 
Smart components have a big responsibility there: **They orchestrate, and know about the codebase!** This means they consume different
parts of the code base and call functions on them, consume state from them, etc.
But that's kind of a big responsibility, to orchestrate with the rest of the application, it should know all
the services and more importantly, have access to them. That might not be the best idea.
If we use the **enforceModuleBoundaries** rule we can make sure that the smart components can't access what they want, but that
does not abstract away anything.

In the following image we see 3 feature libraries that consume whatever they want from within the code base:

- The red boxes are smart components
- The orange boxes are ui components
- The rest are injectables that can be consumed wherever we want

![anarchy](/assets/smart-components-ui-components-and-sandbox-facades-in-angular/anarchy.png)

This is dangerous and can result in anarchy. We don't want our smart components to consume whatever they want in the entire codebase.
The same goes for kids. we don't want them to access everything in our garden, there might be dangerous tools lying around there.
When we put our kids in a sandbox in that garden, they know they can play with little buckets, shovels, and other toys that their sandbox has. The responsibility of our kid is now limited and he or she is safe and can prosper by doing what he
or she is meant to do. We as a parent, shouldn't worry about them taking our lawn mower or hedge saw.

The same goes for smart components: We want to keep them ignorant in a way too.
We don't want them to inject services from wherever they want in the code base. We don't want them to call state actions or selectors.
Smart components shouldn't even know how the state is being managed in the codebase, and they definitely shouldn't know which state management framework the codebase uses.
They shouldn't even know if we use a state management framework.

A sandbox facade will help with that. A sandbox facade is a facade that does not contain any logic. The only thing
that it does is expose functionality for one specific feature lib.

It's an injectable that only exposes the functions and observables that a feature would need.
It would become the **API** of that feature, and it would be very easy to see, what that feature can consume from the rest of the codebase.
The feature would use its sandbox to consume things outside the feature, and thus would be sandboxed in a way.

Every feature would have **one and only one sandbox facade** and the code in there could be seen as redundant. 
That's totally fine because this facade
**can not contain any logic**. It can only contain proxy logic.
The goal is that the feature library can not consume **anything** unless it is exposed through its sandbox facade.

This is illustrated in the following image:
![Sandbox facades](/assets/smart-components-ui-components-and-sandbox-facades-in-angular/sandbox.png)


### What can a sandbox facade do?

- It can expose observables from state services, or even use state selectors
- It can expose public functions from services and orchestrate
- It can send actions on an NGRX store but the method name should not reveal if we are using NGRX.

```typescript
@Injectable()
export class GoodSandboxFacade {
  // GOOD: Pass through observables direcly
  public readonly users$ = this.authService.user$;
  // GOOD: Pass through observables through initial function
  public readonly countries$ = this.masterDataState.getCountries();
  // GOOD: Query parts of state management framework
  public readonly chatboxOpen$ this.store.select(selectChatbox());
  // GOOD: Pass through public functions from certain services
  public addUser(user: User): Observable<User>{
    return this.userService.add(user);
  }
  // GOOD: Sandbox facade abstracts the
  // state management framework away
  public storeUser(user: User): void{
    this.store.dispatch(new addUserAction(user))
  }
}
```

### What can't a sandbox facade do?

- It can not hold state, it **can** expose state by passing it from somewhere else
- It can not contain any RxJS operators (that would be logic, no `map()`, no `shareReplay(1)`)
- It can not call multiple calls in one function: It should only expose things

```typescript
@Injectable()
export class BadSandboxFacade {
  // BAD: Sandbox facade can not hold state, only pass it through
  public readonly user$ = new BehaviorSubject<User>(null);
  // BAD: Sandbox facade can not contain logic, not even map functionality
  public readonly countries$ = this.masterDataState.getCountries().pipe(
    map(resp => resp.elements)
  );
  // BAD: Custom selectors here is also logic
  public readonly chatboxOpen$ this.store.select(() => createSelector(...)));
  // BAD: constructing objects is also logic, only pass things through
  public addUser(firstName: string, lastName: string): Observable<User>{
    return this.userService.add({firstName, lastName});
  }
  // BAD: Sandbox facade can not contain logic, also no navigation logic
  public addCourse(course: Course): Observable<Course> {
    return this.courseService.add(course).pipe(
      tap(() => this.router.navigate('../'))
    )
  }
  // BAD: Sandbox facade can not contain logic, also no state management logic
  public addBook(book: Book): void {
    this.bookService.add(book).subscribe(book => this.store.dispath(...))
  }
  // BAD: Sandbox facade can not contain specific orchestration logic
  // this would be redundant and the urge of reusing would be too big
  public addLearningSuite(learningSuite: LearningSuite)
      : Observable<{book: Book, course: Course}> {
    return combineLatest({
      book: this.bookService.add(learningSuite.book),
      course: this.courseService.add(learningSuite.course)
    })
  }
}
```

Why can't it do those things? Because that would result in the urge to share sandbox facades and that would introduce
a whole new level of complexity. Sandboxes would lose their purpose. Having logic in there would result in redundant logic,
and that is not what we want.

The goal of this facade is to make the smart components ignorant as well. It's there to abstract the rest of the codebase away
from a feature library.

A sandbox facade is the playground of a feature library. It provides a tool belt of all the things a smart component in that
feature can play with.

### Benefits of Sandbox facades

- Smart components have a clear API of tools that they can consume
- The responsibility of the smart components becomes smaller
- Clean constructors or `inject` statements in our smart components
- The presentation layer (feature libraries) can easily be developed when the rest of the code base can be mocked
- We could refactor from one state management framework to another without touching the presentation layer
- It makes developers think about which code they can expose and what code they shouldn't expose
- It would allow huge teams to work in one codebase without interfering with each other if they stick to their sandbox.

## Summary

The single responsibility pattern and decoupling code are 2 important aspects of good software architecture. It will make the purpose of your elements in the
code even clearer and that can help us to reason about code even better.
Ui components have a very clear responsibility. It's rendering the DOM correctly and notifying their direct parent when they need to.
Smart components can orchestrate and feed their child components but by keeping them ignorant we limit their responsibility as well. This would result in
readable code and would make refactoring in the future a breeze.
We can leverage that by sandboxing the feature lib by a sandbox facade.

There are multiple ways of doing things and there is no right or wrong.

### Special Thanks!

Special thanks to the reviewers!

- [Martin](https://twitter.com/chaos_monster){:target="_blank"}
- [Gregor Woiwode](https://twitter.com/GregOnNet){:target="_blank"}
