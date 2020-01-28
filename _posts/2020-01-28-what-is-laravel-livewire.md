---
layout: post
title: What is Laravel Livewire? Here's same app created with Laravel Livewire and pure Vue.js
---

If you ever created full-stack application with reactive frontend and AJAX based data exchange with backend, you know the drill, you:
* create an endpoint which returns JSON on resources
* create specific event handler on frontend which sends AJAX request to backend to fetch resources
* create frontend component which renders resources or handles user input

<!--more-->

Backend and frontend they both maintain they own state, properties, methods, they are not connected in any way, you need to come up with implementation to "glue" them together. This is where Livewire comes place. Main goal of Livewire is to simplify these steps for creating Vue components that exchange data with backend. You just create your components in backend, Livewire handles reactivity for you. To showcase how it works, I decided to create same task management application with Livewire and also with pure Vue.js approach.

Source code of this demo project is available on [Github](https://github.com/orkhanahmadov/livewire-todo).

**Let's see the differences!**

In both cases we have one root `Tasks` component which acts as a wrapper and holds all other related components.
* In Livewire example, it is: `resources/views/livewire/tasks.blade.php`
* In Vue example, `resources/js/Tasks.vue`

## Listing tasks
### Pure Vue.js approach
In Vue example we have `resources/js/TaskList.vue` component for listing all available tasks. To fetch all tasks from backend first we create GET endpoint and controller method for it.

*routes/web.php*
``` php
Route::get('/tasks', [TasksController::class, 'index']);
```

*app/Http/Controllers/TasksController.php*
``` php
public function index(): JsonResponse
{
    $tasks = Task::orderBy('completed_at')->orderByDesc('id')->get();

    return response()->json([
        'incompleteTasks' => $tasks->filter(fn (Task $task) => is_null($task->completed_at))->values(),
        'completeTasks' => $tasks->filter(fn (Task $task) => !is_null($task->completed_at))->values(),
    ]);
}
```
**Note:** I'm using [arrow functions](https://www.php.net/manual/migration74.new-features.php#migration74.new-features.core.arrow-functions) (short closures) feature of PHP 7.4. If you are not familiar with it, I highly recommend taking a look at them and overall 7.4 [changelog](https://www.php.net/releases/7_4_0.php).

Next, we create `fetchTasks()` method in `resources/js/Tasks.vue` component to fetch tasks and group them, which gets executed on component's `mounted` lifecycle hook.
``` js
data: () => ({
    incompleteTasks: [],
    completeTasks: [],
}),

mounted() {
    this.fetchTasks()
},

methods: {
    async fetchTasks() {
        const response = await fetch('/tasks', {
            method: 'GET',
            credentials: 'same-origin',
            headers: {
                'Accept': 'application/json'
            }
        })

        const { incompleteTasks, completeTasks } = await response.json()

        this.incompleteTasks = incompleteTasks
        this.completeTasks = completeTasks
    }
}
```
Then, we pass `incompleteTasks` and `completeTasks` to `TaskList` component as property which renders them to styled task list.

*Tasks.vue*
``` vue
<task-list :incomplete-tasks="incompleteTasks"
           :complete-tasks="completeTasks"
/>
```

*TaskList.vue*
``` vue
<label v-for="task in incompleteTasks"
    :key="task.id"
>
    <input type="checkbox">

    <span>{{ task.name }}</span>
</label>

<hr />

<label v-for="task in completeTasks"
     :key="task.id"
>
    <input type="checkbox" checked>

    <span>{{ task.name }}</span>

    <span>{{ task.completed_at }}</span>
</label>
```

### Livewire approach
First, we create Livewire component using `php artisan make:livewire task-list` command. This command generates `TaskList.php` Livewire component class which is by default placed in `app/Http/Livewire`.
We define `$incompleteTasks` and `$completeTasks` properties.
``` php
class TaskList extends Component
{
    public Collection $incompleteTasks;
    public Collection $completeTasks;

    public function render(): View
    {
        return view('livewire.task-list');
    }
}
```
**Note:** Another feature of PHP 7.4, here I'm using [typed-properties](https://www.php.net/manual/migration74.new-features.php#migration74.new-features.core.typed-properties).

Next, we create `fetchTasks()` method which fetches all tasks from data storage and sets `incompleteTasks` and `completeTasks`. Just like Vue components, Livewire also has its [lifecycle hooks](https://laravel-livewire.com/docs/lifecycle-hooks/) and you can use them to set values or call methods. We use `mount()` lifecycle method to call `fetchTasks()` function.
``` php
public function mount(): void
{
    $this->fetchTasks();
}

public function fetchTasks(): void
{
    $tasks = Task::orderBy('completed_at')->orderByDesc('id')->get();

    $this->incompleteTasks = $tasks->filter(fn (Task $task) => is_null($task->completed_at));
    $this->completeTasks = $tasks->filter(fn (Task $task) => !is_null($task->completed_at));
}
```

When we create Livewire components using `php artisan make:livewire task-list` command, it creates 2 files:
* one is `TaskList.php` component class where we added our properties and methods in above examples
* second one, `task-list.blade.php` view file in `resources/views/livewire`

Livewire components are defined as Blade templates, you can use full power of Blade: directives, loops, partials, conditional rendering, etc. Livewire components cast any public property which is defined in component class to blade variable. This means, inside `task-list.blade.php` view `$incompleteTasks` and `$completeTasks` properties are available for us to use. We render tasks with Blade's `@foreach` directive.
``` html
@foreach($incompleteTasks as $task)
<label>
    <input type="checkbox">

    <span>{{ $task->name }}</span>
</label>
@endforeach

@foreach($completeTasks as $task)
<label>
    <input type="checkbox" checked>

    <span>{{ $task->name }}</span>

    <span>{{ $task->completed_at }}</span>
</label>
@endforeach
```

When opened in browser, Livewire automatically creates Vue instance and "converts" our Blade template to Vue template.

## Creating new task
### Pure Vue.js approach
First, we create backend endpoint which accepts incoming POST request with `task` request body defined. We create new task and return 201 HTTP response code.

*routes/web.php*
``` php
Route::post('/tasks', [TasksController::class, 'store']);
```

*app/Http/Controllers/TasksController.php*
``` php
public function store(Request $request): Response
{
    Task::create(['name' => $request->input('task')]);

    return response()->noContent(201);
}
``` 

Next, we create `resources/js/CreateTask.vue` Vue component, which only has input form element as template.
``` vue
<input placeholder="New task…"
       v-model="task"
       @keydown.enter="store" />
```

Then, we create `task` model property for 2-way data binding and `store()` method which gets executed and sends new task to backend when we press Enter button inside input element.
``` js
data: () => ({
    task: ''
}),

methods: {
    async store() {
        await fetch('/tasks', {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/json',
                'Accept': 'application/json',
                'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').getAttribute('content')
            },
            body: JSON.stringify({task: this.task})
        })

        this.task = ''
    }
}
```

After creating new task, we also need to update task list in `Tasks` component with newly created task item. One way of doing this is emitting event from `CreateTask` component and listening to it in `Tasks` component.

*CreateTask.vue*
``` js
methods: {
    async store() {
        . . .
        this.$emit('update-list')
    }
}
```

*Tasks.vue*
``` html
<create-task @update-list="fetchTasks()" />
```

### Livewire approach
Again, we create new Livewire component using `php artisan make:livewire create-task` and add `$task` public property to hold new task.
``` php
class CreateTask extends Component
{
    public string $task = '';

    public function render(): View
    {
        return view('livewire.create-task');
    }
}
```

Next we add input form element to `create-task.blade.php` view file.
``` html
<input placeholder="New task..." />
```

Livewire components also support 2-way data binding just like Vue's `v-model`, but with `wire:model` directive. Property we pass to this directive must exist in component's class and needs to be publicly accessible. In our case, it is `task`.
``` html
<input placeholder="New task..."
       wire:model="task" />
```

But unlike `v-model`, Livewire components are directly "connected" to backend component class. This means, when we attach `wire:model` to input form element, component will send AJAX request to backend with each keypress in input element. If we don't want this behavior, we can debounce user inputs.
``` html
<input placeholder="New task..."
       wire:model.debounce.500ms="task" />
```

Livewire components also support event handing with `wire:{event-name}` just like Vue's `v-on`. In our example, we want to listen to event when user presses Enter button inside input form element. 
``` html
<input placeholder="New task..."
       wire:model.debounce.500ms="task"
       wire:keydown.enter="store" />
```

In above example we define that when user pressed enter button inside input form element we are going to execute `store` method. Same as Livewire component properties, method names we pass to Livewire event handlers must exist in component's class and needs to be publicly accessible. Next, we create `store()` method in our component class.
``` php
public function store(): void
{
    Task::create(['name' => $this->task]);
    $this->task = '';
}
```

Last thing, after creating new task we need to notify our `TaskList` component that task list is updated. We can use `emit()` method in our component class to fire event with given name. When opened in browser Livewire creates global event bus and all Livewire components in same page/view can listen events fired from it.

``` php
public function store(): void
{
    . . .
    $this->emit('updateList');
}
```

To catch `updateList` event, we go back to `TaskList.php` component class and add `$listeners` property. This property needs to an array, where key is name of the event and value is name of the local, publicly accessible method that needs to be invoked. Since we already have `fetchTasks()` method that fetches all tasks and groups them, we can use this method for updating task list.
``` php
. . .

protected $listeners = [
    'updateList' => 'fetchTasks'
];

. . .
```

When component receives `updateList` event, it will call `fetchTasks()` method, which will fetch all tasks and re-render Blade component and send back updated HTML to frontend instance of Livewire.

## Marking and deleting tasks
If you take a look at [source code](https://github.com/orkhanahmadov/livewire-todo), it also includes additional features like marking tasks as complete/incomplete, deleting tasks.

## So, what's the benefit of using Livewire components?
I guess you can already see how Livewire helps with creating data driven components. Here are few key differences:

* When we create Vue components we always created dedicated REST endpoints to get/create/delete/update resources. But with Livewire it's not needed, every Livewire component automatically creates its own REST endpoints and handles them behind the scenes.
* We created separate methods, event handlers both on frontend and backend.
	* When fetching tasks we called `fetchTasks()` method from `mounted()` lifecycle hook in Vue component. Then we created dedicated controller `index()` method to accept incoming AJAX request, fetched tasks and returned them as JSON.
	* When creating new task, we established local 2-way data binding in Vue component with `v-model`; then attached event handler, called `store()` method on Enter button press; method sent AJAX request to backend with `v-model` value. On backend site, we created dedicated controller `store()` method which created new task and returned success HTTP code.
	* But in all Livewire examples we created properties, method, event listeners only in component class and attached everything in Blade file. All the heavy-lifting, like wiring Blade templates to Vue templates, handling events, sending requests and updating DOM handled by Livewire for us.
* Unit testing becomes super easy. Because everything is in PHP and component properties, methods are publicly accessible, unit testing of components does not require any additional testing tools. Everything can be tested with PHPUnit. In fact, Livewire provides [test helpers](https://laravel-livewire.com/docs/testing/) for unit and end-to-end testing of components.

## Conclusion
Livewire is not a total replacement for frontend libraries or frameworks. Because of its unique reactive nature, every binding, method call or event handling is round-trip to backend. If you want to have full control over how frontend and backend behaves separately or if your frontend component does not rely on data exchange with backend, creating pure Vue components would be better option than using Livewire. But if you want to create frontend component which actively needs to communicate with backend, Livewire makes it super easy to archive.

Also, Livewire does not force you to use Livewire components only, it plays nice with existing Vue components. In same page can mix Livewire components with pure Vue components, Livewire can even [interact with existing pure Vue components](https://laravel-livewire.com/docs/vuejs/).

Livewire has ton of other useful features like [CSS transitions](https://laravel-livewire.com/docs/css-transitions), [input validation](https://laravel-livewire.com/docs/input-validation/), [computed properties](https://laravel-livewire.com/docs/computed-properties/), [resource authorization](https://laravel-livewire.com/docs/authorization/), [polling](https://laravel-livewire.com/docs/polling), [loading states](https://laravel-livewire.com/docs/loading-states/). You can even make [Single-Page-Application using Livewire](https://laravel-livewire.com/docs/spa-mode/).

Project is in active development, new features being added every month, if not every week. You can take a loot at [current roadmap](https://laravel-livewire.com/podcasts/ep27-current-roadmap/) and listen project's [podcasts](https://laravel-livewire.com/podcast).