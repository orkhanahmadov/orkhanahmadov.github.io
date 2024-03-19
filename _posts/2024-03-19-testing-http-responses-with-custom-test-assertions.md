---
layout: post
title: Testing HTTP responses with custom test assertions
---

Laravel's HTTP test assertions are great for testing the response of your application.
They provide a lot of methods to assert the response status, headers, content, and more.

But sometimes you have a specific response that you want to test and there's no built-in method for it, especially when there's a repeating pattern.

<!--more-->

Imagine this endpoint with a response that flashes a message to the session with a redirect:

```php
final class UpdateProfileController
{
    public function __invoke(): RedirectResponse {
        // updates profile

        return redirect()->route('profile')->with('flash', 'Successfully updated!');
    }
}

Route::post('/profile', UpdateProfileController::class)->name('updateProfile');
```

To display the flash message you are probably using a component on the frontend that relies on the `flash` session key with a specific message.
This means you need to make sure all flash messages are consistent and have the correct `flash` session key.

This controller on its own is easy to test:

```php
final class UpdateProfileControllerTest extends TestCase
{
    public function testSuccessfullyUpdatesAndRedirects(): void
    {
        $this->post($this->route, ['foo' => 'bar'])
            ->assertRedirect(route('profile'))
            ->assertSessionHas('flash', 'Successfully updated!');
    }
}
```

But if you have a lot of controllers that flash messages to the session, you can't repeat this assertion in every test case.
It would have been nice if we could do something like:

```php
final class UpdateProfileControllerTest extends TestCase
{
    public function testSuccessfullyUpdatesAndRedirects(): void
    {
        $this->post($this->route, ['foo' => 'bar'])
            ->assertRedirectsWithFlash(route('profile'), 'Successfully updated!');
    }
}
```

This assertion abstracts the redirect and flash message assertion and also makes the test case more expressive.
But if you try to run this test case, you'll get an error that the `assertRedirectsWithFlash` method does not exist.

Test response macros to the rescue!

If you take a look at the source code of `assertRedirect` or `assertSessionHas` you'll see that they are defined on the `\Illuminate\Testing\TestResponse` class.
And if you take a look at its traits, you'll see that it uses `\Illuminate\Support\Traits\Macroable` trait. Which means we can easily add our own custom methods to it.

In the test environment, the easiest way to register custom macros in Laravel is using the `CreatesApplication` trait in the application `tests` directory.

Trait should have `createApplication` method that looks like this:

```php
public function createApplication()
{
    $app = require __DIR__ . '/../bootstrap/app.php';

    $app->make(Kernel::class)->bootstrap();

    return $app;
}
```

Here, just before we return the `$app`, we can register our custom macros that will be only available in the test environment:

```php
public function createApplication()
{
    $app = require __DIR__ . '/../bootstrap/app.php';

    $app->make(Kernel::class)->bootstrap();

    \Illuminate\Testing\TestResponse::macro('assertRedirectsWithFlash', function (string $url, string $message) {
        $this->assertRedirect($url);
        $this->assertSessionHas('flash', $message);
    });

    return $app;
}
```

Now if you run the same test case again, it should pass.
You can use the `assertRedirectsWithFlash` method in any test case that returns the `\Illuminate\Testing\TestResponse` response and makes flash message assertions more expressive and consistent.

Happy testing!
