---
layout: post
title: Testing for the existence of controller middleware in Laravel without invoking them
---

If you are feature-testing your controllers, you’ve probably written test cases like this:

```php
public function testRequiresAuthState(): void
{
    $this->get(route('dashboard'))
        ->assertRedirect(route('login'));
}
```

Here you intentionally don't create/assign any user to the session and try to access a route that requires authentication.

<!--more-->

While they get the job done, when you have a lot of controllers that use the same middleware check it gets repetitive.
You can’t skip writing them as they ensure you don’t mistakenly leave any controller public.

One option to solve this would be creating a custom test assertion method on the main `TestCase` class, something like:

```php
public function assertRouteRequiresAuth(string $route, string $method = 'get'): void
{
    $this->call($method, $route)
        ->assertRedirect(route('login'));
}
```

This method can take a route and method, internally do the assertion. While this solves the previous problem of checking if the route is auth-protected, we can’t write this kind of custom assertion method for every middleware.
Besides, real-world controllers have more than one middleware and usually depend on complex setups. Reproducing those setups on every test case is not ideal.

Take this web controller as an example:

```php
Route::post('/', CreatePostController::class)
    ->middleware(['auth', 'verified', 'role:editor', 'permission:create post'])
    ->name('createPost');
```

This controller uses middleware to make sure the user
- signed in
- has verified email
- has the role of editor
- has permission to post

Additionally, because it is a web controller it needs to have `web` middleware on it too.
How do we test against this controller and assert that all middleware are assigned to it? As we said writing and running dedicated test cases is not an option, they are repetitive, boring, and hard to set up...

But here’s the trick. Since all controllers get registered within Laravel once the framework boots up, all assigned middleware also get registered.
This means we can query Laravel's route registrar for a specific route with a name and get the list of assigned middleware to it!
Here's how we can do it:

```php
$route = \Illuminate\Support\Facades\Route::getRoutes()->getByName($name);

$assignedMiddleware = $route->gatherMiddleware();
```

We can create a custom test assertion method on `TestCase`:

```php
protected function assertRouteHasMiddleware(string $name, array $expectedMiddleware): void
{
    $route = Route::getRoutes()->getByName($name);
    
    if (is_null($route)) { // in case non-existing route name is passed
        $this->fail("Route `{$name}` not defined.");
    }

    $this->assertSame(
        $expectedMiddleware,
        $route->gatherMiddleware(),
        "Expected middleware are not assigned to the route `{$name}`".
    );
}
```

Using the previous example, we can now test the `CreatePostController` like this:

```php
public function testCreatePostControllerMiddleware(): void
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

That's it! Now we can test for the existence of middleware without actually invoking the controllers.
This is a great way to make sure that your controllers are properly protected without writing repetitive test cases.

This is especially important when you are using middleware from Laravel itself or third-party packages.
You shouldn't be testing those middleware's actual behavior, but assume that they are already unit tested, and you should only test that they are assigned to the controller that we use.

Of course, this approach also assumes that you are already unit-testing your application's custom middleware. For example, if `role:editor` is a custom middleware, you should cover it with unit tests too.

Happy testing!
