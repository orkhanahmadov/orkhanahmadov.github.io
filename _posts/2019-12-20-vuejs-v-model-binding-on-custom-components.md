---
layout: post
title: Vue.js v-model binding on custom components
---

When working with Vue.js `v-model` directive comes very handy for 2-way data binding with form element.
But when creating custom components with form elements in them it becomes hard to maintain this 2-way binding with parent component.

Image we have `InputElement.vue` component with input form element in it.
``` js
<template>
    <div>
        <input type="text" class="some-class" placeholder="Input something" />

        <span>Helper text here</span>
    </div>
</template>
```

And it is used in some other component like this:

``` js
<template>
    <form>
        <label>Input your name</label>

        <input-element />
    </div>
</template>

<script>
import InputElement from '~/components/InputElement'

export default {
    components: {
        InputElement
    }
}
</script>
```

Because input element is inside child component it parent component does not have access to input value.

But if closely how `v-model` works behind the scenes, this directive basically does 2 things when applied to element:
* Binds data to elements `value` attribute
* Catches `oninput` events fired from element

This means we can replace any input `v-model` binding with `:value="someProperty"` and `@input=handleInput($event)`:

``` js
<input v-model="someProperty" /> // this is same as

<input :value="someProperty" 
       @input="someProperty = $event.target.value"
/> // this one
```

Of course when using `v-model` Vue.js is smart enough to bind values and events based on element type.
For example, `v-model` automatically detects that when used on `<select>` element it should look for `onchange` event, instead of `oninput`.

But now knowing that how `v-model` works bind the scenes, 
we can use the same technique to attach `v-model` directive to a custom component,
get `value` as property inside that component and bind it input value.

Here's our new `InputElement.vue` component
``` js
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

``` js
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

To notify `v-model` about value changes inside our component we need to fire `input` element

``` js
<template>
    <div>
        <input type="text"
               class="some-class"
               placeholder="Input something"
               :value="value"
               @input="$emit('input', $event)" />

        <span>Helper text here</span>
    </div>
</template>

<script>
export default {
    props: ['value']
}
</script>
```

Here `@input` gets event from native `<input>` form element, then we emit our `input` event to parent component with initial `$event` payload.

Now that we satisfied `v-model`'s with our value and event binding we establish for 2-way binding with custom component.

Keep in mind that when attaching `v-model` to custom components, by default it always sends `value` property and expects `input` event.
But luckily Vue.js components allow you change this behavior.
Simply add `model` object to your component. This object can contain to properties:
* `prop` property defines how `v-model` will pass data, default is `value`
* `event` property defines how `v-model` will accept changes from component, default is `input`

``` js
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
it will automatically send values as `typed` property and accept updated values from `changed` event.