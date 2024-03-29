---
layout: post
title: Unit testing exceptions in PHP
---

Let's imagine we have the following code that runs and conditionally throws an exception:

```php
use Symfony\Component\HttpKernel\Exception\HttpException;

class DeletePostAction
{
    public function execute(Post $post)
    {
        if ($post->is_published) {
            throw new HttpException(403, 'Cannot delete a published post');
        }

        $post->delete();
    }
}
```

If we want to write test coverage for this class it will require at least 2 test cases:

- Assert that the action deletes the provided post successfully
- Assert that the action throws an exception when the provided post is published

<!--more-->

The first one is straightforward:

```php
public function testDeletesPostWhenNotPublished(): void
{
    $post = new Post(is_published: false);

    $action = new DeletePostAction();
    $action->execute($post);

    $this->assertFalse($post->exists()); // post should be deleted
}
```

What about the second one? We want to assert that the action throws an exception when the post is published.
For starters, we can use PHPUnit's `expectException` method to do this:

```php
public function testThrowsExceptionWhenPostIsPublished(): void
{
    $post = new Post(is_published: true);

    $action = new DeletePostAction();

    $this->expectException(HttpException::class);

    $action->execute($post);
    $this->assertTrue($post->exists()); // post did not get deleted
}
```

Here we instruct PHPUnit to expect an instance of `HttpException` to be thrown when the action is executed and this test will pass without any issues.

But if you noticed we are not asserting the exception's message, we are only asserting that an exception is thrown.
This can be solved by using the `expectExceptionMessage` method:

```php
public function testThrowsExceptionWhenPostIsPublished(): void
{
    $post = new Post(is_published: true);

    $action = new DeletePostAction();

    $this->expectException(HttpException::class);
    $this->expectExceptionMessage('Cannot delete a published post');

    $action->execute($post);
    $this->assertTrue($post->exists());
}
```

We are done with the message, but if we go back to the exception that we throw in the `DeletePostAction` class, we also need to assert the status code of the exception.
We use `HttpException` and it can have different status codes. Here we throw that exception exactly with 403 status code, so we need to assert that the exception has the same status code.
You might think for this we can use the PHPUnit's `expectExceptionCode` method, like:

```php
public function testThrowsExceptionWhenPostIsPublished(): void
{
    $post = new Post(is_published: true);

    $action = new DeletePostAction();

    $this->expectException(HttpException::class);
    $this->expectExceptionMessage('Cannot delete a published post');
    $this->expectExceptionCode(403);

    $action->execute($post);
    $this->assertTrue($post->exists());
}
```

But if you run it, you'll see the test case is not passing and complaining about the exception code `0` not the same as `403`.
Why is that? Because PHPUnit's `expectExceptionCode` checks against the exception's internal code, not the HTTP status code.
In this case for `HttpException` it is by default `0` and we are not modifying the exception's internal code.

If we take a look at `HttpException` you can see that the provided `$statusCode` is saved as a private property and there is a method `getStatusCode` to get it:

```php
class HttpException extends \RuntimeException implements HttpExceptionInterface
{
    private int $statusCode;
    private array $headers;

    public function __construct(int $statusCode, string $message = '', ?\Throwable $previous = null, array $headers = [], int $code = 0)
    {
        $this->statusCode = $statusCode;
        $this->headers = $headers;

        parent::__construct($message, $code, $previous);
    }

    public function getStatusCode(): int
    {
        return $this->statusCode;
    }

    ...
}
```

When you need to check against all the possible parameters and properties of the exception, we can no longer use PHPUnit's `expectException` or `expectExceotionMessage` methods.
Instead, we can use a `try-catch` block inside the test case and do the assertions manually.

```php
public function testThrowsExceptionWhenPostIsPublished(): void
{
    $post = new Post(is_published: true);

    $action = new DeletePostAction();

    try {
        $action->execute($post);
        $this->assertTrue($post->exists());
        $this->fail('Expected exception was not thrown');
    } catch (HttpException $e) {
        $this->assertSame(403, $e->getStatusCode());
        $this->assertSame('Cannot delete a published post', $e->getMessage());
    }
}
```

In this case, we expect that the test case execution must go into the `catch` block, and if it does not we fail the test case manually with a `fail()` call.

This gets the job code, now we have a full test coverage for the action class.

But looking at this test case and seeing how we do a workaround to assert the exception, I believe it highlights a possible improvement to our code.
The exception we are throwing in our code is no longer a basic `HttpException`, but a modified version of it with a custom message and status code.
This is a good opportunity to create a custom exception class for this use case and abstract the exception details into it. Something like:

```php
use Symfony\Component\HttpKernel\Exception\HttpException;

class CannotDeletePublishedPostException extends HttpException
{
    public function __construct()
    {
        parent::__construct(403, 'Cannot delete a published post');
    }
}
```

Now we can use it in our action class:

```php
class DeletePostAction
{
    public function execute(Post $post)
    {
        if ($post->is_published) {
            throw new CannotDeletePublishedPostException();
        }

        $post->delete();
    }
}
```

Lastly, we can go back to our test case and use the custom exception class to assert the exception:

```php
public function testThrowsExceptionWhenPostIsPublished(): void
{
    $post = new Post(is_published: true);

    $action = new DeletePostAction();

    $this->expectException(CannotDeletePublishedPostException::class);

    $action->execute($post);
    $this->assertTrue($post->exists());
}
```

As you can see here, because we use the custom exception class, we no longer need to assert against the internals of this exception class.
PHPUnit's `expectException` method is enough to assert that the exception is thrown and the test case passes.

Of course, we still want to be sure that `CannotDeletePublishedPostException` is using the correct status code and message,
but it is no longer the concern of this test case, but rather the concern of a dedicated test case for the exception class itself.

```php
class CannotDeletePublishedPostExceptionTest extends TestCase
{
    public function testStatusCodeAndMessage(): void
    {
        $exception = new CannotDeletePublishedPostException();

        $this->assertSame(403, $exception->getStatusCode());
        $this->assertSame('Cannot delete a published post', $exception->getMessage());
    }
}
```
