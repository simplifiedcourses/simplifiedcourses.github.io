---
layout: post
title: "Extracting forms complexity into ViewModels"
date: 2023-11-19
published: false
cover: assets/extracting-template-driven-forms-complexity-into-viewmodels.jpg
comments: true
categories: [Angular, Angular Forms, Angular Signals]
description: "In this article, we will lean how to extract Angular Forms functionality into ViewModels"
---

# Intro

This article is about using ViewModels to ensure clean templates for Template-driven Forms.
[Rather see the YouTube video](todo){:target="_blank"}?
For a personal project I'm working on a solution to create recurring tasks.
We can have a task that only executes once (eg on Black Friday),
but we can also have tasks that repeat every day, or every few days
or certain days every week, certain days every few weeks and so on...
For this solution we used the [Template-driven Forms solutio I shared before](https://blog.simplified.courses/i-opensourced-my-angular-template-driven-forms-solution/){:target="_blank"}.

The type of our form looks like this:

```typescript
export type RecurringTaskFormModel = Partial<{
  name: string; // name of the task
  description: string; // description of the task
  executeOn: string; // The day when the task should be executed
  executeOnTime: string; // The time when the task should be executed
  repeat: boolean; // Should this task be recurring
  repetitionSettings: Partial<
    {
      repeatEveryAmount: number; // Repeat every x (days or weeks)
      repetitionType: 'day'|'week'; // Repeat every so many (day or week)
      repeatEveryWeekValues: number[]; // When week, which weekdays do we want to repeat
    }
  >
}>;
```

The form has a bunch of functionality to make this work:
- Only show the `repetitionSettings` when the task is recurring (`repeat` to `true`).
- When the `repeatEveryAmount` is set to `1`, singular labels should be shown in the `repetitionType`.
  field, when the value is higher than `1` plural labels should be shown in the `repetitionType` field.
- When the `repetitionType` is set to `week`, show the `repeatEveryWeekValues`.

### Template logic

Putting that logic in the template isn't the ideal way of doing things. Template-driven Forms are awesome
because Angular does all the work for us... But we want to keep our templates clean.

Having these `*ngIf` statements in the template is template logic, and it shatters
our all over the DOM, which is unwanted. Having `*ngIf` directives is fine of course, but we want to keep away
the template-logic as much as possible.

The contents in these `*ngIf` directives is what we want to avoid:

```html
<div ngModelGroup="repetitionSettings" *ngIf="formValue?.repeat">
    <h3>Repetition settings</h3>
    ...
    <div *ngIf="formValue?.repetitionSettings?.repetitionType === 'week'">
        <select name="repeatEveryWeekValues"...>
        </select>
    </div>
</div>
```

It even becomes worse if we want to solve the singular/plural problem where we want to show
`week` if the value of `repeatEveryAmount` is set to `1` and `weeks` when its higher than `1`:

```html
<label scControlWrapper>
    <span>Repeat every type</span>
    <select type="number" name="repetitionType"
            [ngModel]="formValue?.repetitionSettings?.repetitionType">
        <option [value]="option.value"
                *ngFor="let option of repetitionTypeOptions; trackBy: tracker">
            {%raw%}{{
                formValue?.repetitionSettings?.repeatEveryAmount === 1 ?
                option.label :
                option.labelMultiple
            }} {%endraw%}
        </option>
    </select>
</label>
```

### ViewModels

I've written and talked about ViewModels before, but in this scenario in combination with Template-driven Form components
they really shine.

In our recurring tasks form let's create a signal from our input:

```typescript
export class RecurringTasksFormUiComponent {
    private readonly recurringTaskForm = signal<RecurringTaskFormModel>({});

    @Input()
    public set formValue(v: RecurringTaskFormModel) {
        this.recurringTaskForm.set(v);
    };
}
```

All the logic mentioned before can be calculated in the viewModel which:
- gives us a nice centralized place where the magic happens
- leeps our template clean
- avoids potential redundancy
- is declarative, reactive and readable

```typescript
export class RecurringTasksFormUiComponent {
    private readonly recurringTaskForm = signal<RecurringTaskFormModel>({});

    @Input()
    public set formValue(v: RecurringTaskFormModel) {
        this.recurringTaskForm.set(v);
    };

    // Declarative, reactive, readable, simple computed signal that contains all the complexity
    private readonly viewModel = computed(() => {
        const form = this.recurringTaskForm();
        return {
            formValue: form,
            showRepetitionSettings: form.repeat,
            showExecuteOnTime: form.repeat,
            repetitionTypeOptions: this.repetitionTypeOptions.map(option =>
                ({
                    value: option.value,
                    label: (form?.repetitionSettings?.repeatEveryAmount || 0) > 1 ? option.labelMultiple : option.label
                })),
            showRepeatEveryWeekValues: form?.repetitionSettings?.repetitionType === 'week'
        };
    });

    protected get vm() {
        return this.viewModel()
    }
    ...
}
```

When creating this reactive ViewModel for us, our first snipped cleaned up very nice!
It now looks like this:

```html
<div ngModelGroup="repetitionSettings" *ngIf="vm.showRepetitionSettings">
    <h3>Repetition settings</h3>
    ...
    <div *ngIf="vm.showRepeatEveryWeekValues">
        <select name="repeatEveryWeekValues"...>
        </select>
    </div>
</div>
```

There is no more logic in our template anymore. It all got moved to our clean ViewModel.
The second snippet now looks like this:

```html
<label scControlWrapper>
    <span>Repeat every type</span>
    <select type="number" name="repetitionType"
            [ngModel]="vm.formValue?.repetitionSettings?.repetitionType">
        <option [value]="option.value"
                *ngFor="let option of vm.repetitionTypeOptions; trackBy: tracker">
            {%raw%}{{ option.label }}{%endraw%}
        </option>
    </select>
</label>
```

We do the translation of multiple and singular labels in our ViewModel, resulting in a nice and clean template.

### Conclusion

This was a short but important article because it contains a Best Practice

**Angular Template-Driven Forms can be awesome but don't put your actual logic in the template**
Why not?
- It's harder to test
- The DOM is there for semantic reasons, not to contain actual functional javascript code
- It's reactive and declarative to put it in the ViewModel

Hope you enjoyed this!