---
layout: post
title: Using Laravel migrations to modify data in tables during deployments
---

There are many ways to modify data in a table during deployment.
Some applications use database seeds, some use console commands and some use completely custom scripts with raw SQL queries.

But in Laravel, there's a easiest and most convenient way to do this: using migrations.
You've probably done it several times. You wanted to modify data in a table during deployment and you used a migration to do it.

It gets the job done, doesn't require any additional setup/script/library, and is version controlled by default.

<!--more-->

But using migrations for data modifications in tables also comes with some downsides.

- It's not the primary purpose of migrations. Migrations are meant to modify the structure of the database, not the data.
- It bloats the migrations directory
- Since version 8, Laravel offers [migration squashing](https://laravel.com/docs/10.x/migrations#squashing-migrations), which is a great feature to squash and remove old migrations files. But migration squashing relies on actual database structure, it does not create SQL statements for data modifications, but only for the structure.

But because migrations are so convenient and easy to use, can we use them to modify data in tables during deployments without these downsides?

Yes, the answer is using different folders to store data modification migrations.

Simply, create a new folder in the `database` directory, name it `data-migrations`, for example. Then add all your data modification migrations in this folder.
Since data migrations are usually one-way only, you can also skip the `down` method in these migrations for simplicity.

How do we run these migrations? Laravel's migration command always looks for migrations in the `database/migrations` directory, but it is possible to instruct to look into different directory for migration files.

```shell
php artisan migrate --path=database/data-migrations
```

This command will run all migrations in the `database/data-migrations` directory.

This approach solves all the previous downsides of using migrations for data modifications in tables. It keeps the migrations directory clean and focused on database structure modifications, and when you want to squash migrations, you can do it without worrying about data migrations.
Since data migrations are in a separate directory, they are only meant for data modifications you can safely delete them after they've been run and no longer needed on any environments.

Of course, because we still use Laravel's migration, every migration is version controlled in `migrations` table.