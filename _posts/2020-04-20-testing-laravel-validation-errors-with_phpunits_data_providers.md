---
layout: post
title: Testing Laravel validation errors with PHPUnit's data providers
---

Laravel testing tools are very powerful for doing endpoint (feature) tests. Let's say we have an endpoint which accepts requests as JSON. Take this simple test as an example:

<!--more-->

```php
public function testCreatesCustomer()
{
    $response = $this->postJson('customers', [
        'full_name' => 'John Doe',
        'email' => 'john@example.com',
    ]);

    $response->assertCreated();
}
```

If we have this kind of endpoint, we probably also have request validation for it, like following:

`App\Http\Controllers\CustomerController.php`
```php
class CustomerController
{
    public function store(CustomerStoreRequest $request)
    {
        // creates customer

        return response()->noContent(201);
    }
}
```

`App\Http\Requests\CustomerStoreRequest.php`
```php
class StoreRequest extends FormRequest
{
    public function rules()
    {
        return [
            'full_name' => ['required', 'string', 'max:30'],
            'email' => ['required', 'email', 'unique:customers:email'],
            'phone' => ['nullable', 'integer'],
        ];
    }
}
```

Our test case will pass because we provide all required fields and all in a valid format.

But how we can test if endpoint requires "full_name" field or "email" value should not already exist in the database.
One way we can archive this by creating test cases for each validation rule.

```php
public function testRequiresFullName()
{
    $response = $this->postJson('customers', []);

    $response->assertStatus(422);
    $response->assertJsonValidationErrors('full_name');
    $this->assertSame(
        \Lang::get('validation.required'),
        $response->json('errors.full_name.0')
    );
}
```

This approach is fine, but when we have many fields and validation rules, 
it becomes hard to write and maintain all test cases for each validation rule.

Instead, we can use the power of PHPUnit's data providers.
We can create data provider function with all input and output and let PHPUnit repeat same test case with different values.
Here's our data provider function:

```php
public function validationErrors()
{
    return [
        [['full_name' => ''], 'full_name', 'validation.required'],
        [['full_name' => \Str::random(31)]], 'full_name', 'validation.string.max', ['max' => 30]],
        [['email' => ''], 'email', 'validation.required'],
        [['email' => 'not-an-email'], 'email', 'validation.email'],
        [['email' => 'existing@example.com'], 'email', 'validation.unique'],
        [['phone' => 'abc'], 'phone', 'validation.integer'],
    ];
}
```

In this data provider:
* First argument is request data that we want PHPUnit to submit, e.g. we send empty "full_name" `['full_name' => '']`
* Second argument is the validation field we expect to fail, e.g. when we send empty "full_name", we except that "full_name" field would exist in validation errors
* Third argument is validation message that we expect to be shown, e.g.  when we send empty "full_name" we expect that translation of `validation.required` message will be shown
* Fourth and optional argument is special parameters that validation message requires, e.g. when we send "full_name" with 31 characters we expect that error message with "30" as maximum value will be shown.

Here's our test case that uses `validationErrors()` method as data provider:

```php
/**
 * @dataProvider validationErrors
 */
public function testValidationErrors(
    array $invalidData,
    string $invalidField,
    string $errorMessage,
    array $messageParams = []
) {
    factory(Customer::class)->create(['email' => 'existing@example.com']);
    
    $response = $this->postJson('customers', $invalidData);

    $response->assertStatus(422);
    $response->assertJsonValidationErrors($invalidField);
    $this->assertTrue(\Lang::has($errorMessage));
    $this->assertSame(
        \Lang::get(
            $errorMessage,
            array_merge(
                ['attribute' => str_replace('_', ' ', $invalidField)],
                $messageParams
            )
        ),
        $response->json("errors.{$invalidField}.0")
    );
}
```

What's happening here?
* First, as preparation we create a customer with the same email address that we used in our data provider for uniqueness test.
* We send a request to our endpoint with provided `$invalidData`
* We assert that response came back from the endpoint has 422 HTTP code
* We assert that provided `$invalidField` exists in JSON validation errors
* We do extra check by verifying if provided `$errorMessage` is valid, exists and translatable key
* Lastly, we assert that provided translation of `$message` with `$messageParams` is matching with the validation error field from the endpoint. Translation of generic Laravel validation rules requires "attribute" as a translation parameter and if the field key contains "_", validation rule converts it to empty space. Here we do the same in our expected message field.

Now if we run `testValidationErrors()` test, it will run itself over and over (6 times in our case) with all data sets and 
output an error if any data set does not succeed.
With this, now we can easily test Laravel validation errors, without creating single test cases for each validation rule.

Additionally, you can move the assertion part of this test case somewhere accessible from all test classes, to a parent `TestCase` class maybe. 
Or even better approach would be creating your assertion macro on `Illuminate\Testing\TestResponse`.