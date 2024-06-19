---
layout: post
title: "Angular workspace architecture: Stop making everything re-usable"
date: 2024-05-15
published: true
cover: assets/angular-workspace-architecture-stop-making-everything-reusable.jpg
comments: false
authors: [brecht_billiet]
categories: [Angular architecture]
description: "Are you making everything re-usable just in case? Your making a mistake and I will explain why"
---

# Angular workspace architecture: Stop making everything re-usable

## Intro

This article is part of a series of articles I'm writing about Angular Architectures of large-scale workspaces.
I'm creating an all-in-one online training as we speak on how to architect huuuuuuuge codebases. You can check the progress [here](https://www.simplified.courses/largescale-angular-enterprise-architecture){:target="_blank"}
When we are architecting huge workspaces, encapsulation becomes extremely important.
We want to limit developers as much as possible in terms of using existing code, so they don't end up shooting themselves in
the foot.

#### What do we mean by that?

What we mean by that, is that just because there is code available in the workspace, that it does **not** mean that that code should be used everywhere. 
In this article we will tackle the concept of re-usability in depth.

## Do not make code available if you don't need to

This should send a clear message, I know we all learned how to be DRY (don't repeat yourself) in school, and I know once a component or
utility is there, it's great to re-use it and not re-invent the wheel over and over again. 

However, when we make everything re-usable, our big workspace will end up in one huge bowl of pain and become un-manageable in the long-run.
For people that are new to workspaces, a workspace is a solution that contains apps and libs. 
Your apps are empty shells, and your libs contain all the logic.
You can use it for mono-repos or huge projects.
We are using [Nx/devtools](https://nx.dev){:target="_blank"} for this, which is an amazing technology that helps you maintain gigantic projects.

Writing reusable code is a best-practice. **Exposing all code so that it can be reused is NOT!**
When working in huge workspaces with a big number of colleagues, we are all working in the same pond and trying not to interfere with each other. This can be dangerous.
Think about it... Numerous applications, hundreds and hundreds of libs, everybody reusing other people their code.
I have been working with those workspaces since the beginning of Angular, and I can proudly say I farmed workspaces with +300 projects and dozens of applications inside.
It's a beautiful way to collaborate, and we don't need to manage local NPM dependencies, but we do need some ground rules.

### An example where code-reuse can be dangerous

Let's say that our good colleague "Joe" is working on a `visitors` feature in our workspace, and he needs an `address`  component that has 4 address lines for his feature. 

There is no `address` component yet in our workspace, so our good friend Joe here, decides to create an `address`  component in a `shared` folder, because maybe that component could be useful to someone else.
That's very thoughtful of Joe but a few months later our colleague "Jane" that is working on a `customers` feature also needs an `address`  component and finds that awesome component that Joe had created. 

She believes there is a reason this component is shared, so she imports the component blindly and delivers her feature in record time.
The next sprint there is a feature request where Jane her team-mate "Joseph" needs to remove an address line in the `customers` feature.
That doesn't seem like a hard task so Joseph removes the address line, fulfilling his feature request but unintentionally broke the feature of Joe.
The `address` component in the `visitors` feature now only contains 3 address lines, resulting in a bug in the `visitors`feature.
After a while they realise this, and they want to make the `address` adaptable to both cases. 
They implement an `@Input()` property with a conditional `*ngIf` statement that shows or hides the 4th line.
This is a minor change, but now the `address` component got more responsibilities and gotten a little bit more complex.

After a while it starts to derail:
- The `visitors` feature needs countries with a country `flag` in the dropdown and the `customers` don't.
- Oh, the `customers` also need billing information, like the VAT number
- The customers might also need a shipping address part and a billing address part
- ...

I think you get the point... While the `address` component is a very simple example, it already starts to derail. What about more complex components?
What about a `detail-person` component? What about types that are customized for the `visitors` feature? What about state-machines tailored for the `customers` feature?
What if we don't only have the `visitors` and `customers` feature, but have thousands of features in hundreds of libs in dozens of applications, working with potentially hundreds of developers?
How do we make sure those developers don't wind up killing each other? I have one beautiful word for you right here: **ENCAPSULATION**
We make sure that not every developer can import whatever the hell they want in whatever feature they are working in.

Don't get me wrong... There are use-cases where this `address` component needs to be consistent across the workspace.
But, if a component like this address component needs to be reusable, **it should be a decision that is taken in a team meeting**. Re-usability can be dangerous because:
- It creates dependencies.. The more dependencies, the harder your workspace becomes to maintain.
- It can create circular dependencies if not thought about thoroughly.
- It prevents flexibility, because every change impacts all its dependants. Bye bye single responsibility principle.
- It invites for more complexity, because after a while the component will be filled with conditionals, covering all the needs of each and every dependant.

## When should we reuse something?

There are 2 rules of thumb.
The first is easy. **Don't over-engineer.** You can make stuff re-usable at any given time... It's harder to move a re-usable component
to a non-re-usable component. Stay safe, expand later.

The second rule of thumb should be this little quiz (let's take a component for this example):
- If there is no one at this stage in need of that component, keep it unexposed.
- If you know you might need it in the future, but you are not sure, still keep it unexposed. 
- If there is a dependant in need of that component, start a meeting with the product-owner and ask yourselves this:
  - Do we need the exact same component or slightly different behavior?
  - Will the dependant need more features in the future?
  - How much complexity do we need to add to make this component cover all cases?
- Now, If it makes sense to reuse it, expose the component! You have thought it through!
- Otherwise, copy-paste it and give that component its own life. You can always merge it later, but at least you don't have any unneeded dependencies now

##  How to prevent a component from being exposed

When using a workspace framework like Nx/Devtools we tend to create apps and libs.
Every lib has its barrel file (`index.ts`) that is used to expose members of that lib.
So when the component should not be exposed, don't expose it in the barrel file.
Let's think about the `visitors` lib that is a super duper complex lib with hundreds of components, state-machines, services and what not.
Our barrel file should look like this:

```typescript
// index.ts
export { routeConfig } from './src/lib/routes.ts';
```

That's right!! The only thing that the rest of the workspace needs to have access to, is the route config, so it can lazy load this lib.
It should not know about the components or any other stuff in this lib. It should only have access to what is really needed.
This feature lib is now locked down and the rest of the huge workspace is not bothered with it.
When using something like Nx/Devtools the feature imports should look like this:

```typescript
{
    path: 'customers',
    loadChildren: () =>
        import('@simplified/client-feat-customers').then(
            (mod) => mod.routeConfig
        )
    },
{
    path: 'visitors',
    loadChildren: () =>
        import('@simplified/client-feat-visitors').then(
        (mod) => mod.routeConfig
    )
}
```

Both feature libs only expose a route config and nothing else, resulting in a less error-prone solution.

## Where do we put the re-usable code?

There are a lot of use-cases where we DO want to expose re-usable code, and when we have decided a piece of code should be re-usable we definitely want to expose it.
This points us to the next question. Where do we want to put the re-usable code?
Let's take that `address` component again. We start off by creating the `visitors` feature and as a best practice, we keep the `address` component there, since there is no other feature that needs it now.
BOOM! The `customers` feature arrives, and they are in desperate need of the `address` component and the product owner and all the developers happily agreed that the `address`component should be re-usable.
Do we put it in the `customers` feature? Do we put it in the `visitors` feature? Do we create a `shared` scope and put it in there.

While this sound like a challenging task, it's super easy. If a piece of code is used by 2 libs, 99% of the time you should create a third lib.
You have to ask yourself, **Why does it belong in that feature lib more than in the other feature lib?** It doesn't.
You need a third lib. When the codebase is small, you can start with something like `shared-ui-design-system`.
Don't over-engineer from the beginning. It's ok to move stuff around **all the time**. 
After a while more components might be created like the fancy country `flag` dropdown.
In that case we could create a `shared-ui-address` lib for that.

After a while there might be `address` validations, utilities that contain VAT checks and a whole bunch of other stuff.
In that case we will might create an `address` scope that contains the following libs:
- `ui-address`
- `util-addrss`
- ...

Don't be afraid to refactor, be afraid to over-engineer.

## Conclusion

Hey! Don't shoot me! It's ok to re-use stuff. In fact, it's stupid not to re-use stuff.
But take that decision with your team, and don't make it re-usable just because you might need it in the future.
Working in a huge workspace means training yourself to refactor and move things around on a day-to-day basis.
Keep improving that beauty of a codebase and protect yourself and your team mates.
Hope you like the article!

Looking forward to your feedback!
