---
layout: post
title: Logging executed database queries in Laravel
---

There are known ways in Laravel to measure database performance, keep query count to minimum, prevent "N+1" issues.
Tools like [laravel-debugbar](https://github.com/barryvdh/laravel-debugbar) is a must-have to see what's happening with every request.

<!--more-->

Laravel ships with query logging functionality you can use to log and see every database query.
Simply enable query logging before database calls.

```php
\DB::enableQueryLog();
```

Every database call after this like will be logged and stored in memory. To get list of logged queries, use:

```php
\DB::getQueryLog();
```

This will return array of logged queries.

To disable logging, use:
```php
\DB::disableQueryLog();
```

You can use query logging for analyzing database performance, but it is especially useful for counting executed database queries in tests.

Lets imagine this scenario:
* You have `books` and `authors` tables
* Each book belongs to one author
* You want to send request to `/books` endpoint to retrieve all books with their author relationship.

If you use Eloquent and you apply best practices, then realistically doesn't matter the number of records in tables, application should fire exactly 2 database queries:
* One for getting all `books`
* One for getting all `authors` related to those books

Here's our test covers executed database query count:

```php
public function testExecutesExactly2Queries()
{
    factory(Author::class, 5)->create()
        ->each(fn ($author) => factory(Book::class, 5)->create(['author_id' => $author->id]));

    // we enable query logging after factory class,
    // because we don't author and book creations to be logged
    \DB::enableQueryLog();

    $this->get('/books');

    $this->assertCount(2, \DB::getQueryLog());
}
```

This is very simple example, but it should give you the idea how to use query logging in your tests and database performance checks.
Especially, when your application has complex queries, query caching, it is very handy to utilize query logging and counting logged queries.