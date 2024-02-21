---
layout: post
title: "Making Angular template-driven forms type-safe"
date: 2024-01-23
published: true
cover: assets/making-angular-template-driven-forms-type-safe.jpg
comments: true
categories: [Angular Forms, Angular]
description: "In this article, I will show you how to make template-driven forms completely typesafe"
---

**Updated 21 february 2024**

Template-driven forms usually come with a huge productivity boost.
In [this video](https://www.youtube.com/watch?v=ijp_qt3SYl4&t=1s){:target="_blank"} I explain how we can use signals to
create unidirectional template-driven forms in Angular.
We end up with:
- No boilerplate code at all
- Predictable code
- A solution where Angular does all the work for us

Everything is explained in depth in [this article](https://blog.simplified.courses/angular-template-driven-forms-with-signals/){:target="_blank"}.

### There is a caveat though

We ensure type-safety in the class of our component. We also ensure type-safety in the template in the `[ngModel]` directive.
However, where we can't enforce type-safety is in the `name` and `ngModelGroup` attributes.
Take this example for instance:

```html
<form ...>
    <div ngModelGroup="addresses">
        <div ngModelGroup="shippingAddress">
            <input type="text" name="street" ...>
            <input type="text" name="zipCode" ...>
        </div>
        <div ngModelGroup="invoiceAddress">
            <input type="text" name="street" ...>
            <input type="text" name="zipCode" ...>
        </div>
    </div>
</form>
```

It's awesome that Angular will automatically create the following form structure for us:

```typescript
form = {
    addresses: {
        shippingAddress: {
            street: '',
            zipCode: ''
        },
        invoiceAddress: {
            street: '',
            zipCode: ''
        }
    }
}
```

However, let's make a typo in `shippingAddress` and `street`:

```html
<form ...>

    <div ngModelGroup="addresses">
        <div ngModelGroup="sippingAddress">
            <input type="text" name="street" ...>
            <input type="text" name="zipCode" ...>
        </div>
        <div ngModelGroup="invoiceAddress">
            <input type="text" name="streetttt" ...>
            <input type="text" name="zipCode" ...>
        </div>
    </div>
</form>
```

Our model will now look like this:

```typescript
form = {
    addresses: {
        sippingAddress: { // Error: typo
            street: '',
            zipCode: ''
        },
        invoiceAddress: {
            streetttt: '', // Error: typo
            zipCode: ''
        }
    }
}
```

The problem with this is there will be no compilation errors. There is no way to ensure type-safety on this and we
as developers tend to write the occasional typo. This problem will surface at run-time, when the user is using the form.
We would want to see the following errors:
- **[ngModelGroup] Mismatch: 'addresses.sippingAddress'`**
- **[ngModel] Mismatch: '${addresses.invoiceAddress.streetttt}'`**

This would make it instantly clear to the developer where he or she messed up.

### Hooking into the update cycle to validate the template-driven form result

As explained previously in [this article](https://blog.simplified.courses/angular-template-driven-forms-with-signals/){:target="_blank"} we will keep
the form value in a signal of a specific type. For the address form example the type will look like this:

```typescript
export type AddressFormModel = {
    street: string;
    zipCode: string;
}

// Used in the submit method
export type ValidAddressesFormModel = {
    addresses: {
        shippingAddress: AddressFormModel;
        invoiceAddress: AddressFormModel;
    }
}

// Used for the template-driven part
export type AddressFormModel =  DeepPartial<ValidAddressesFormModel>; 
```

Since a template-driven form will be built by Angular on the fly it makes sense to make every form group and form control optional.
For that I have created this handy `DeepPartial` type for you:

```typescript
export type DeepPartial<T> = {
    [P in keyof T]?: T[P] extends Array<infer U>
        ? Array<DeepPartial<U>>
        : T[P] extends ReadonlyArray<infer U>
            ? ReadonlyArray<DeepPartial<U>>
            : T[P] extends object
                ? DeepPartial<T[P]>
                : T[P];
};
```

The solve our problem we need to compare the output of Angular (the template-driven form) to some kind of object.
In the previous article we learned how to use a [form directive](https://blog.simplified.courses/angular-template-driven-forms-with-signals/#form-directive){:target="_blank"}
to easily get access to a `formValueChange` output.
The example below shows where we could validate the result of the template-driven form with some kind of object.

```html
<form (formValueChange)="onFormValueChange($event)">
    ...
</form>
```

```typescript
export class AddressesFormComponent {
    // Signal that holds the form value
    protected readonly formValue = signal<AddressesFormModel>({});
    
    protected onFormValueChange(event: AddressesFormModel): void {
        // update the form
        this.formValue.set(event);
        
        // TODO: Validate some how, but only in development mode
    }
}
```

I have tried the schema validations solution from vest, and I did research to other solutions but as I love to keep
things simple I came up with this solution:

### Shape validations

What kind of problem are we trying to solve? Do we want to validate types? Let's zoom out a bit.
We are trying to validate whether we as developers made typo's or not
That's the only thing we want to know, we want to get a beautiful error when we type `sippingAddress` instead of `shippingAddress`.
We want to get another fancy error when the `t` key gets stuck, and we get `streetttt` instead of `street`.

Let's define a shape where we defined the models for our form:

```typescript
export type AddressFormModel = {
    street: string;
    zipCode: string;
}

export type ValidAddressesFormModel = {
    addresses: {
        shippingAddress: AddressFormModel;
        invoiceAddress: AddressFormModel;
    }
}

...

export const addressFormModelShape: AddressFormModel = {
    street: '',
    zipCode: ''
}

export const addressesFormModelShape: ValidAddressesFormModel = {
    addresses: {
        shippingAddress: {...addressFormModelShape},
        invoiceAddress: {...addressFormModelShape},
    }
}
```

As we can see, we have defined 2 new objects: `addressFormModelShape` and `addressesFormModelShape`.
Since the problem that we are trying to solve is avoiding typo issues, it does not matter what values the properties
of those objects have. Only the name of the properties is important. For that reason we can just use default values:
- `''` for strings
- `0` for numbers
- `true` for booleans

The goal is to compare the `event` in the `onFormValueChange()` method to this shape everytime the Angular framework
updates the template-driven form behind the scenes. We will need to create a `validateShape()` function that will
be used like this:

```typescript
export class AddressesFormComponent {
    // Signal that holds the form value
    protected readonly formValue = signal<AddressesFormModel>({});
    
    protected onFormValueChange(event: AddressesFormModel): void {
        // update the form
        this.formValue.set(event);
        
        // Show errors when there are typos, but only in development mode
        validateShape(event, addressesFormModelShape);
    }
}
```

### Implementing the validate shape functionality

The functionality of this function is quite straightforward:
**For every property that the template-driven form creates, validate if this property exists in the shape**
We need a way to recursively scan the (by Angular) created object and make sure it matches the shape:

```typescript
/**
 * Clean error
 */
export class ShapeMismatchError extends Error {
    constructor(errorList: string[]) {
        super(`Shape mismatch:\n\n${errorList.join('\n')}\n\n`);
    }
}

/**
 * Only validate the shape in dev mode
 * Create ShapeMisMatchError if needed at runtime
 * @param val
 * @param shape
 */
export function validateShape(
    val: Record<string, any>,
    shape: Record<string, any>,
): void {
    if (isDevMode()) {
        const errors = validateFormValue(val, shape);
        if (errors.length) {
            throw new ShapeMismatchError(errors);
        }
    }
}

function validateFormValue(formValue: Record<string, any>, shape: Record<string, any>, path: string = ''): string[] {
    const errors: string[] = [];
    for (const key in formValue) {
        if (Object.keys(formValue).includes(key)) {
            const newPath = path ? `${path}.${key}` : key;
            if (typeof formValue[key] === 'object' && formValue[key] !== null) {
                if ((typeof shape[key] !== 'object' || shape[key] === null) && isNaN(parseFloat(key))) {
                    errors.push(`[ngModelGroup] Mismatch: '${newPath}'`);
                }
                errors.push(...validateFormValue(formValue[key], shape[key], newPath));
            } else if ((shape ? !(key in shape) : true) && isNaN(parseFloat(key))) {
                errors.push(`[ngModel] Mismatch '${newPath}'`);
            }
        }
    }
    return errors;
}
```


### Form array issue

This solution works great! We are able to throw runtime errors in development mode when the developer has mistyped any
`name` or `ngModelGroup` attribute, the rest will get picked up by the compiler.
However, we still have issues when it comes to form arrays. In [this article](https://blog.simplified.courses/template-driven-forms-with-form-arrays/){:target="_blank"}
I explain how to create form arrays with template-driven forms.
Let's update our addresses form example, so it takes a dynamic list of addresses:

```typescript
export type AddressFormModel = {
    street: string;
    zipCode: string;
}

export type ValidAddressesFormModel = {
    addresses: {
        addValue: AddressFormModel;
        values:  { [key: string]: AddressFormModel };
    }
}
```

```html
<form ...>
    <div ngModelGroup="addresses">
        <div ngModelGroup="addValue">
            <input type="text" name="street" ...>
            <input type="text" name="zipCode" ...>
        </div>
        <div ngModelGroup="values">
            <div [ngModelGroup]="key" 
                 *ngFor="let key of formValue.addresses.values | keyvalue; trackBy: tracker">
                <input type="text" name="street" ...>
                <input type="text" name="zipCode" ...>
            </div>
        </div>
    </div>
</form>
```

Great! This will return nicely into an object that looks like this, that will live in the `formValue` signal:

```typescript
form = {
    addresses: {
        addValue: {
            street: '',
            zipCode: ''
        },
        values: {
            0: {
                street: '',
                zipCode: ''
            },
            1: {
                street: '',
                zipCode: ''
            },
            2: {
                street: '',
                zipCode: ''
            }
        }
    }
}
```

We see that we have 3 addresses in here. So how do we create a shape for that?
Let's try creating a shape:

```typescript
export const addressFormModelShape: AddressFormModel = {...}

export const addressesFormModelShape: ValidAddressesFormModel = {
    addresses: {
        addValue: {...addressFormModelShape},
        addresses: {
            0: {...addressFormModelShape}
        }
    }
}
```

That seems legit, however what happens if we have multiple addresses? There is no way of knowing how many addresses
there will be, and we want to validate everything.
To fix this we can update the `validateFormValue()` function and add this clause:
If the key is a number (means it is an index of an array) and the number is bigger than 0 we should always validate
towards the `0` key in that shape. This means that **for form arrays, we always need to fill in one item in our shapes**,
just like we did in the example:

Here is the updated `validateFormValue()` function:

````typescript
function validateFormValue(formValue: Record<string, any>, shape: Record<string, any>, path: string = ''): string[] {
    const errors: string[] = [];
    for (const key in formValue) {
        if (Object.keys(formValue).includes(key)) {
            // In form arrays we don't know how many items there are
            // so every time reset the key to '0' when the key is a number and is bigger than 0
            let keyToCompareWith = key;
            if(parseFloat(key) > 0){
                keyToCompareWith = '0';
            }
            const newPath = path ? `${path}.${key}` : key;
            if (typeof formValue[key] === 'object' && formValue[key] !== null) {
                if ((typeof shape[keyToCompareWith] !== 'object' || shape[keyToCompareWith] === null) && isNaN(parseFloat(key))) {
                    errors.push(`[ngModelGroup] Mismatch: '${newPath}'`);
                }
                errors.push(...validateFormValue(formValue[key], shape[keyToCompareWith], newPath));
            } else if ((shape ? !(key in shape) : true) && isNaN(parseFloat(key))) {
                errors.push(`[ngModel] Mismatch '${newPath}'`);
            }
        }
    }
    return errors;
}
````

Did you know I open-sourced the entire solution [here](https://blog.simplified.courses/i-opensourced-my-angular-template-driven-forms-solution/){:target="_blank"}?
You can play with it on stackblitz etc.

### Conclusion

Template-driven forms are awesome, but they are prone to typo's in templates.
The typescript compiler takes care of everything except the `name` and `ngModelGroup` attributes.
By creating simple shapes for our form models and validating them in the development process,
we can improve DX drastically by throwing runtime errors.

Hope you enjoyed the article!
If you liked it, please leave a comment or share!
