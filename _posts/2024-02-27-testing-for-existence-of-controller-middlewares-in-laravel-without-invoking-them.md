---
layout: post
title: Testing for existence of controller middlewares in Laravel without invoking them
---

If you are feature testing your controllers, you’ve probably written test cases like this:

```php
public function testRequiresAuthState(): void
{
    $this->get(route('dashboard'))
        ->assertRedirect(route('login'));
}
```

Here you intentionally don't create/assign any user to session and try to access a route that requires authentication.

<!--more-->

While they get the job done, when you have a lot of controllers that use the same middleware check it gets repetitive.
You can’t also skip writing them as they make sure you don’t mistakenly leave any controller public.

One option to solve this would be creating custom test assertion method on the main `TestCase` class, something like:

```php
public function assertRouteRequiresAuth(string $route, string $method = 'get'): void
{
    $this->call('get', $route)
        ->assertRedirect(route('login'));
}
```

This method can take a route and method, internally do the assertion. While this solves the previous problem for checking if route is login protected, we can’t write this kind of custom assertion methods for every single middleware.
Besides, real-world controllers has more than one middleware and they usually depend on complex setup. Repeating that setup on every test case is not ideal.

Take this web controller as an example:

```php
Route::post('/', CreatePostController::class)
    ->middleware(['auth', 'verified', 'role:editor', 'permission:create post'])
    ->name('createPost');
```

This controller uses middlewares to make sure user
- signed in
- has verified email
- has role of editor
- has permission to post

Additionally, because it is a web controller it needs to have web middleware on it too.
How do we test against this controller and assert that all middlewares assign to it. Writing and running dedicated test cases it not an option, they are repetitive, boring and hard to setup...

But here’s the trick. Since all controllers get registered within Laravel once framework boots up, all assigned middlewares also get registered.
This means we can query Laravel's route registrar for specific route and get the list of assigned middlewares to it!
Here's how we can do it:

```php
$route = \Illuminate\Support\Facades\Route::getRoutes()->getByName($name);

$assignedMiddlewares = $route->gatherMiddleware();
```

We can create a custom test assertion method that does this for us:

```php
protected function assertRouteHasMiddleware(string $name, array $expectedMiddlewares): void
{
    $route = Route::getRoutes()->getByName($name);
    
    if (is_null($route)) { // in case non-existing route name is passed
        throw new RouteNotFoundException("Route [{$name}] not defined.");
    }

    $this->assertSame(
        $expectedMiddlewares,
        $route->gatherMiddleware(),
        "Expected middlewares are not assigned to the route `{$name}`".
    );
}
```

Using the previous example, we can now test the `CreatePostController` like this:

```php
public function testCreatePostControllerMiddlewares(): void
{
    $this->assertRouteHasMiddleware('createPost', [
        'web',
        'auth',
        'verified',
        'role:editor',
        'permission:create post',
    ]);
}
```

That's it! Now we can test for the existence of middlewares without actually invoking the controllers.
This is a great way to make sure that your controllers are properly protected without writing repetitive test cases.

This is especially important when you are using middlewares from Laravel itself or from third-party packages.
You shouldn't be testing those middlewares actual behavior, you should assume that they are already unit tested, and you should only test that they are assigned to the controller that we use them on.

Of course, this approach also assumes that you are already unit testing your application's custom middlewares. For example, if `role:editor` is a custom middleware, you should cover them with unit tests too.

Happy testing!
