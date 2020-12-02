---
layout: post
title: Rendering response from Laravel exception
---

In your Laravel controller or somewhere in your code you probably have a try/catch like this:

```php
use App\Exceptions\FailedRequestException;

class ApiController
{
    public static sendRequest()
    {
        $api = new SomeApi();
        
        try {
            $api->sendRequest();

            return response()->json([
                'data' => $api->getData(),
            ]);
        } catch(FailedRequestException $e) {
            return response()->json([
                'error' => 'Failed to send API request',
            ]);
        }
    }
}
```

<!--more-->

In this example we simply send a request to some API and return data from API response. If request throws `FailedRequestException` exception we catch and return error message instead.

In Laravel if we already use dedicated exception class, manually catching that exception to return response is not necessary.
If we take a look into default application exception handler in `App\Exceptions\Handler`, it has `render()` method which runs parent method.
That parent `render()` method contains this code:

```php
public function render($request, Throwable $e)
{
    if (method_exists($e, 'render') && $response = $e->render($request)) {
        return Router::toResponse($request, $response);
    } elseif ($e instanceof Responsable) {
        return $e->toResponse($request);
    }
    
    ...
}
```

Here we can see that whenever exception gets thrown in Laravel, 
exception handler before rendering exception first checks if exception class has `render()` method in it or exception class is an instance of `Responsable` interface.
If any of these checks are satisfied Laravel will render contents of `render()` method on exception class instead of rendering default exception view.

Knowing this new trick we can modify our example `FailedRequestException` class to have `render()` method.

```php
class FailedRequestException extends \Exception
{
    public function render($request)
    {
        return response()->json([
            'error' => 'Failed to send API request',
        ]);
    }
}
```

Or we have use `Responsable` interface to archive same result

```php
use Illuminate\Contracts\Support\Responsable;

class FailedRequestException extends \Exception implements Responsable
{
    public function toResponse($request)
    {
        return response()->json([
            'error' => 'Failed to send API request',
        ]);
    }
}
```

Both `render()` and `toResponse()` methods must accept single request parameter, exception handler will pass instance of `Illuminate\Http\Request` to it.

Now going back to our try/catch example, we no longer need to catch `FailedRequestException` exception only to return custom response

```php
class ApiController
{
    public static sendRequest()
    {
        $api = new SomeApi();
        $api->sendRequest();
        
        return response()->json([
            'data' => $api->getData(),
        ]);
    }
}
```

Whenever application throw `FailedRequestException` exception Laravel's exception handler will catch it and render our custom response.