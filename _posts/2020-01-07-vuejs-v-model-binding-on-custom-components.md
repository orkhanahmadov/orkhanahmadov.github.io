---
layout: post
title: Vue.js v-model binding on custom components
---

When working with form elements, Vue's `v-model` directive comes very handy for 2-way data binding.
But when you create your own custom form components and wrap input elements inside, 
it becomes hard to maintain same 2-way binding with custom form component and parent component.
<!--more-->

### Problem

Imagine we have `InputElement.vue` component with input form element in it.
``` vue
<template>
    <div>
        <input type="text"
               placeholder="Your name" />

        <span>Helper text here</span>
    </div>
</template>
```

And it is used in some other component like this:

``` vue
<template>
    <form>
        <label>Enter your name</label>

        <input-element />
    </div>
</template>

<script>
import InputElement from '~/components/InputElement'

export default {
    components: {
        InputElement
    },

    data() {
        return {
            name: ''
        }
    }
}
</script>
```

Imagine we want to bind `name` data property to input element, 
but actual input element is wrapped inside `<input-element>` component, not directly accessible to parent component.

Luckily, Vue.js allows us use `v-model` on custom components to establish 2-way input binding.
But for this, first we have to look closely how `v-model` actually works.

### How `v-model` works

In short, this directive basically does 2 things when applied to form element:
* Binds reactive property to element's `value` attribute
* Handles `oninput` events fired from element to get new value

This means we can replace any input `v-model` directive with `:value` property binding and `@input` event handler:

``` vue
// this is
<input v-model="someProperty" />

// same as this one
<input :value="someProperty" 
       @input="someProperty = $event.target.value"
/>
```

Of course, when working with native html form elements like `input`, `select` or `textarea`, 
it is always better to use `v-model` instead of manual binding.
Because `v-model` is shorter, simpler to implement and 
also Vue.js is smart enough to bind values and events based on type of form element.
For example, `v-model` automatically detects that when used on `<select>` element 
it should look for `onchange` event, instead of `oninput`.

Now knowing that how `v-model` works behind the scenes, 
we can use the same technique to add `v-model` directive to custom component,
inside the component we'll get `value` as property and bind it to input value.

Here's our new `InputElement.vue` component
``` vue
<template>
    <div>
        <input type="text"
               class="some-class"
               placeholder="Input something"
               :value="value" />

        <span>Helper text here</span>
    </div>
</template>

<script>
export default {
    props: ['value']
}
</script>
```

Updated parent component that uses `InputElement.vue`:

``` vue
<form>
    <label>Input your name</label>

    <input-element v-model="name" />
</div>

<script>
import InputElement from '~/components/InputElement'

export default {
    components: {
        InputElement
    },

    data() {
        return {
            name: ''
        }
    }
}
</script>
```

To notify `v-model` about value changes inside our component we need to fire `input` event every time value changes.

``` vue
<template>
    <div>
        <input type="text"
               class="some-class"
               placeholder="Input something"
               :value="value"
               @input="$emit('input', $event.target.value)" />

        <span>Helper text here</span>
    </div>
</template>

<script>
export default {
    props: ['value']
}
</script>
```

Here `@input` handler looks for `oninput` event fired from native `<input>` form element,
then we fire our own `input` event to parent component with new input value as payload.

Now that we satisfied `v-model` directive with our value and event binding, 
we establish for 2-way binding with custom component.

### Customizing default value and event names

Keep in mind that when attaching `v-model` to custom components, 
by default it always sends `value` property and expects `input` event.
But Vue.js allows us to change default names per each component.

Simply add `model` object to your component. This object can contain to properties:
* `prop` property defines how `v-model` will pass data to this component, default is `value`
* `event` property defines which event name will be used for accepting changes for `v-model`, default is `input`

Here's our `InputElement.vue` component with custom model value and event name:  
``` vue
<template>
    <div>
        <input type="text"
               class="some-class"
               placeholder="Input something"
               :value="typed"
               @input="$emit('changed', $event)" />

        <span>Helper text here</span>
    </div>
</template>

<script>
export default {
    model: {
        prop: 'typed',
        event: 'changed'
    },
    props: ['typed']
}
</script>
```

In this example when `v-model` added to `<input-element>` component,
it will automatically send values as `typed` property and will expect updated values from `changed` event.

### Usage tips

Learning this small trick with `v-model` directive opens ton of possibilities for establishing 
2-way data binding with custom components.

Don't be limited to just form elements. 
`v-model` does not care how you handle passed `value` property or when you fire `input` event back.
You can create custom components which doesn't even have any form elements, 
but based on component's behavior, it can handle property and event on its own.