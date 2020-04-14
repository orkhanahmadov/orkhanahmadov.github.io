---
layout: post
title: Accessing parent and child components in Vue.js
---

In Vue.js creating nested components and passing data between them is pretty straight forward thanks to property binding and event emitting. Take this as an example.

<!--more-->

`Tasks.vue` component:
```vue
<template>
    <div>
        <h1>Task list</h1>

        <div v-for=“task in tasks”>
            <task-item :task=“task” @refresh=“refreshList” />
        </div>
    </div>
</template>

<script>
export default {
    data: () => ({
        tasks: []
    }),

    mounted() {
        this.refreshList()
    }

    methods: {
        refreshList() {
            // send HTTP request to get fresh task list
        }
    }
}
</script>
```

`TaskItem.vue` component:
```vue
<template>
    <div>
        <h2>{{ task.name }}</h2>
        <small>{{ task.description }}</small>

        <a href=“#” @click.prevent=“markAsCompleted(task)”>Complete</a>
        <a href=“#” @click.prevent=“deleteTask(task)”>Delete</a>
    </div>
</template>

<script>
export default {
    props: [‘task’],

    methods: {
        markAsCompleted(task) {
            // send HTTP request to backend mark task as completed

            this.$emit(‘refresh’)
        },

        deleteTask(task) {
            // send HTTP request to the backend to delete the task

            this.$emit(‘refresh’)
        }
    }
}
</script>
```

Now imagine we want to move task item actions to its component, something like `TaskActions.vue`
```vue
<template>
    <div>
        <a href=“#” @click.prevent=“markAsCompleted(task)”>Complete</a>
        <a href=“#” @click.prevent=“deleteTask(task)”>Delete</a>
    </div>
</template>

<script>
export default {
    props: [‘task’],

    methods: {
        markAsCompleted(task) {
            // send HTTP request to backend mark task as completed

            this.$emit(‘refresh’)
        },

        deleteTask(task) {
            // send HTTP request to the backend to delete the task

            this.$emit(‘refresh’)
        }
    }
}
</script>
```

Updated `TaskItem.vue`:
```vue
<template>
    <div>
        <h2>{{ task.name }}</h2>
        <small>{{ task.description }}</small>

        <task-actions :task=“task” />
    </div>
</template>

<script>
import TaskActions from ‘~/components/TaskActions’

export default {
    props: [‘task’],

    components: {
        TaskActions
    }
}
</script>
```

If you noticed, with this approach now we have a problem. `this.$emit()` call is no more on `TaskItem` component but `TaskActions`. Events fired from `this.$emit()` can be listened only on the direct parent component. In our case, the parent of `TaskActions` component is `TaskItem` component, this means `@refresh=“refreshList”` listener will no longer work on `Tasks` component.

## Meet `$parent` property
Every Vue component has a special property called `$parent` which allows us to access direct parent component from the current child component. When using `$parent` acts as you are on the parent component, this means you can access parent component’s properties, data, fire actions, mutate data also emit and listen to the event. Knowing this, we can now refactor our `TaskActions` component to emit events on parent component instead, with `this.$parent.$emit()`

Updated `TaskActions.vue`
```vue
<template>
    <div>
        <a href=“#” @click.prevent=“markAsCompleted(task)”>Complete</a>
        <a href=“#” @click.prevent=“deleteTask(task)”>Delete</a>
    </div>
</template>

<script>
export default {
    props: [‘task’],

    methods: {
        markAsCompleted(task) {
            // send HTTP request to backend mark task as completed

            this.$parent.$emit(‘refresh’)
        },

        deleteTask(task) {
            // send HTTP request to the backend to delete the task

            this.$parent.$emit(‘refresh’)
        }
    }
}
</script>
```

Because the parent of the `TaskActions` component is `TaskItem`, events will be emitted on `TaskItem` component, this time `Tasks` component will be able to catch our “refresh” event.

## What about accessing child components?
Vue also provides `$children` property, when used returns array of direct child components of the current component. Because `children` property returns an array of child components and without any order guarantee, this makes it a little bit hard to work with.
The better option is assigning “reference” to child components and accessing them through `$refs` property. Here’s an example:

`Parent.vue`
```vue
<template>
    <div>
        <h1>Title</h2>

        <my-component-1 ref=“myReferencedComponent” />
        <my-component-2 />

        <button type=“button” @click.prevent=“handleClick”>Button</button>
    </div>
</template>

<script>
import MyComponent1 from ‘~/components/MyComponent1’
import MyComponent2 from ‘~/components/MyComponent2’

export default {
    components: {
        MyComponent1,
        MyComponent2
    },

    methods: {
        handleClick() {
            this.$refs.myReferencedComponent.doSomething()
        }
    }
}
</script>
```

`MyComponent1.vue`
```vue
<template>
    <div>
        Lorem ipsum
    </div>
</template>

<script>
export default {
    methods: {
        doSomething() {
            //
        }
    }
}
</script>
```

As you can see from our example if we assign `ref` attribute to our child component after component gets successfully mounted that reference becomes available as a property of `$refs` object. Now accessing `this.$refs.myReferencedComponent` acts as we are in the child `MyComponent1` component - we can access its data, methods, properties, etc.

## Caveats
Accessing parent component with `$parent` and child component with `$refs` needs to be done carefully as the usage of them adds complexity and dependency to components, sometimes they also introduce unexpected behavior. For example, by using `$parent` on `TaskActions`  we made this component dependent on some unknown parent property which needs to have `refreshList()` method. This means we can no longer use this component in a different place where the parent component does not have this method or method exists but it does not behave the same way. Also, doing something on parent/child component which mutates reactive data can cause unexpected behavior. Use them very carefully.