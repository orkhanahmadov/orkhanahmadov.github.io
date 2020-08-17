---
layout: post
title: Using traits to boot and initialize Eloquent models
---

If you ever used Eloquent events, you are probably aware of special `boot()` static method in Eloquent models.
This method allows you to hook into special events by running given functions.

Here's an example, let's say we have a `Post` model and when we are creating a new post, model needs to generate slug `attribute` based on `name`.

<!--more-->

```php
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

class Post extends Model
{
    public static function boot()
    {
        parent::boot();
    
        static::creating(function (Model $model) {
            $model->slug = Str::slug($model->name);
        });
    }
}
```

**Quick tip:** Since Laravel 7, we can also use `booted()` static method, it is same as `boot()` method, but using `booted()` means we no longer need to call parent method.

Now, here's a situation, let's say we also have `Author` model and that model also needs to have exactly same behavior, it needs to generate author slug based on an author's `name` attribute.
We don't want to duplicate same code piece on `Author` model, but how we can share the same logic between multiple independent models?

One way to archive this would be creating custom base abstract model class, something like `ModelWithSlug`, putting logic inside and extending this class from both `Post` and `Author` models.
This might look fine for single shared logic like slug generation, but what if we want to have different shared logic together with slug generation, like UUID generation.
For example, `Post` models need to generate UUID on creation but this does not apply to `Author` model.
This will kill the purpose of having our own base class, since we'll need to override that `boot()` method again for `Post` model.

There's a better solution. In PHP it easy to share same piece of code between multiple classes using traits.
But, without overriding `boot()` method we don't have control over how it works, and we cannot call our trait methods.

### Introducing "bootable" Eloquent traits

Let's take a look at `boot()` method on `Illuminate\Database\Eloquent\Model` class works.

```php
protected static function boot()
{
    static::bootTraits();
}
```

`boot()` method just calls another static method `bootTraits()`, let's see what that one does.

```php
protected static function bootTraits()
{
    $class = static::class;

    $booted = [];

    static::$traitInitializers[$class] = [];

    foreach (class_uses_recursive($class) as $trait) {
        $method = 'boot'.class_basename($trait);

        if (method_exists($class, $method) && ! in_array($method, $booted)) {
            forward_static_call([$class, $method]);

            $booted[] = $method;
        }

        if (method_exists($class, $method = 'initialize'.class_basename($trait))) {
            static::$traitInitializers[$class][] = $method;

            static::$traitInitializers[$class] = array_unique(
                static::$traitInitializers[$class]
            );
        }
    }
}
```

Few things happening here.
1. Function loops over all traits that model uses
2. Prefixes trait's base name with `boot` and looks if method with that name exists
3. If method exists, then forwards static call to execute it

This gives us the following idea.
If we create a trait with name `MyTrait` and put a static function in it with name `bootMyTrait` Eloquent will execute this function automatically whenever models gets booted.

Here's our new `HandlesSlug` trait:

```php
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

trait HandlesSlug
{
    public static function bootHandlesSlug()
    {
        static::creating(function (Model $model) {
            $model->slug = Str::slug($model->name);
        });
    }
}
```

Here are `Post` and `Author` models:

```php
use App\Models\HandlesSlug;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HandlesSlug;
}

class Author extends Model
{
    use HandlesSlug;
}
```

Now this approach will solve our original problem by having UUID generation on `Post` model without affecting `Author` model.
We can create another trait like `HandlesUuid`:

```php
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

trait HandlesUuid
{
    public static function bootHandlesUuid()
    {
        static::creating(function (Model $model) {
            $model->uuid = Str::uuid();
        });
    }
}
```

Now we add this trait to `Post` model alongside `HandlesSlug`:

```php
use App\Models\HandlesSlug;
use App\Models\HandlesUuid;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HandlesSlug;
    use HandlesUuid;
}
```

That's all. Everything looks clean and shared logic can be attached to any model without creating complexity or unnecessary inheritance. 
Eloquent will handle booting each trait.

### Wait, what about that `initialize` thing in `bootTraits()`?

Other than `bootMyTrait()` method, we can also have `initializeMyTrait()` in your Eloquent traits.
"Initializer" method needs to be non-static, and it will get executed when new model gets instantiated.

Here's an example:

```php
use Illuminate\Support\Str;

trait CreatesApiKey
{
    public function initializeCreatesApiKey()
    {
        $this->api_key = Str::random();
    }
}
```

Now, whenever you create a new model which uses `CreatesApiKey` trait, Eloquent will automatically call `initializeCreatesApiKey()` method and assign `api_key` attribute.
Initializer trait methods are useful when you want to generate and assign some value to an attribute only when model gets instantiated, 
and value of that attribute doesn't need to depend on another model attribute.  