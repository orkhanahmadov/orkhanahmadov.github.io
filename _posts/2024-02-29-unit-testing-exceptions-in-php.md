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

If we want to write test coverage for this class it will require 2 test cases:

- One to assert that the action deletes the provided post
- Another to assert that the action throws an exception when the provided post is published

<!--more-->

First one, is straightforward:

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
}
```

Here we instruct PHPUnit to expect an instance of `HttpException` to be thrown when the action is executed and this test will pass without any issues.

But if you noticed we are not asserting the message of the exception, we are only asserting that an exception is thrown.
This can be solved by using the `expectExceptionMessage` method:

```php
public function testThrowsExceptionWhenPostIsPublished(): void
{
    $post = new Post(is_published: true);

    $action = new DeletePostAction();

    $this->expectException(HttpException::class);
    $this->expectExceptionMessage('Cannot delete a published post');

    $action->execute($post);
}
```

We are done with the message, but we go back to the exception that we throw in the `DeletePostAction` class, we also need to assert the status code of the exception.
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
}
```

But if you run it, you'll see test case is not passing and complaining about the exception code `0` not the same as `403`.
Why is that? Because PHPUnit's `expectExceptionCode` checks against exception's internal code, not the HTTP status code.
In this case for `HttpException` it is by default `0` and we are not modifying the exceptions internal code.

If we take a look at `HttpException` you can see provided `$statusCode` is saved as a private property and there is a method `getStatusCode` to get it:

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
Instead, we can use a `try-catch` block inside the test case and do the assertions manually:

```php
public function testThrowsExceptionWhenPostIsPublished(): void
{
    $post = new Post(is_published: true);

    $action = new DeletePostAction();

    try {
        $action->execute($post);
    } catch (HttpException $e) {
        $this->assertSame(403, $e->getStatusCode());
        $this->assertSame('Cannot delete a published post', $e->getMessage());
    }
}
```

This gets the job code, now we have a full test coverage for your action class.

But looking at this test case and seeing how we do a workaround to assert the exception, this highlights a possible improvement to our code.
The exception we are throwing in our code is no longer a basic `HttpException`, but modified version of it.
We modify its status code and message to fit exactly into our business logic.
This is good opportunity to create a custom exception class for this use case and abstract the exception details into it. Something like:

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
}
```

As you can see because we use the custom exception class, we no longer need to assert against the internals of this exception class.

Of course, we still want to be sure that `CannotDeletePublishedPostException` is using the correct status code and message,
but it is no longer the concern of this test case, but rather the concern of dedicated test case for the exception class itself.

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