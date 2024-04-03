---
layout: post
title: Testing scheduled commands and jobs in Laravel's console kernel
---

Your Laravel application's console kernel probably contains some scheduled tasks.
But how do you make sure they are configured correctly and no developer will accidentally remove or modify them?

Take this console kernel as an example:

```php
class Kernel extends ConsoleKernel
{
    protected $commands = [];

    protected function schedule(Schedule $schedule): void
    {
        $schedule->command('prune-stale-files')->hourly();
        $schedule->job(VerifyUserSubscriptions::class)->daily();
    }
}
```

<!--more-->

Here we have an artisan command and a queued job scheduled to run every hour and every day respectively.
Assuming you already tested them individually in unit tests, how do you test the console kernel to make sure they are scheduled correctly?

The easiest way to do this would be to use test macros on Laravel's Scheduler facade.
If you've seen my previous post about [testing HTTP responses with custom assertions in Laravel](https://orkhan.dev/2024/03/19/testing-http-responses-with-custom-assertions-in-laravel/), we can use the same approach here.

We create a macro that checks against the scheduled command:

```php
use Illuminate\Console\Scheduling\Event;
use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Support\Arr;

Schedule::macro('hasCommand', function (string $command, string $expression): bool {
    $event = Arr::first(
        $this->events(),
        fn (Event $item): bool => Str::after($item->command, "'artisan' ") === $command && $item->expression === $expression
    );
    
    return ! is_null($event);
});
```

Here macro accepts the command name and the cron expression and checks if there is an event with the same command and expression registered in the scheduler.

Now we can use this macro in our test case:

```php
use Illuminate\Support\Facades\Schedule;

public function testShouldSchedulePruningStaleFilesDaily(): void
{
    $this->assertTrue(
        Schedule::hasCommand('prune-stale-files', '0 0 * * *')
    );
}
```

How about the queued job? We can create another, dedicated macro for that:

```php
Schedule::macro('hasJob', function (string $job, string $expression): bool {
    $event = Arr::first(
        $this->events(),
        fn (Event $item): bool => $item->description === $job && $item->expression === $expression
    );

    return ! is_null($event);
});
```

And a similar test case for the queued job:

```php
public function testShouldScheduleVerifyUserSubscriptionsDaily(): void
{
    $this->assertTrue(
        Schedule::hasJob(VerifyUserSubscriptions::class, '0 0 * * *')
    );
}
```
