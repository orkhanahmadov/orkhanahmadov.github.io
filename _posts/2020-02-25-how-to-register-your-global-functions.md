---
layout: post
title: How to register your global functions in PHP using Composer
---

Composer simplifies class autoloading with different techniques and standards. Nowadays most common class autoloading standard is [PSR-4](https://www.php-fig.org/psr/psr-4/):
```json
"autoload": {
    "psr-4": {
        "App\\": "src/"
    }
}
```

<!--more-->

This will autoload all classes inside `src` folder using PSR-4 standard with "App" namespace prefix.
But how can we autoload files, global helper functions that are not directly part of namespaced classes?
Well, if you take a look at [official documentation](https://getcomposer.org/doc/04-schema.md#autoload) on Composer website, you can see that "autoload" schema supports multiple standards and techniques.
* `PSR-0` is old autoload standard, deprecated, but still supported. We should avoid using it.
* `PSR-4` is modern autoload standard replaced `PSR-0`. It is main autoloading standard for namespaced classes
* `classmap` is a autoload standard for loading classes without namespace or namespace prefix
* `files` is a autoload standard for loading files

As you can already guess, `files` is what we need to load PHP files without class definition, containing only helper methods. `files` standard accepts array of relative paths to each file.
Usually, when you want to define global helper methods it is better to create a single PHP file with "functions.php" or "helpers.php" name and put all your helper functions inside it.
```json
"autoload": {
    "files": [
        "src/functions.php"
    ]
}
```
There are some common practices on how to write and autoload global functions. You need to keep in mind that because these functions do not have namespaces when Composer loads them, they can cause conflict with already existing functions. Composer autoload mechanism always checks if given function or class already exists, if it does, Composer throws an exception saying that "cannot redeclare".
To avoid this, before loading a function we can check if it is already available.

*functions.php*
```php
if (!function_exists('sayHello')) {
    function sayHello()
    {
        return 'Hello!';
    }
}
```

In this example, we ask Composer to check if global function with `sayHello` name already exists, it not then load given function, otherwise ignore it and use already available function.