---
layout: post
title: Validating ZIP file content in Laravel
---

When uploading file in Laravel application request validation gives us good support for validating uploaded file. You can define if uploaded file needs be an [image](https://laravel.com/docs/6.x/validation#rule-image), you can set maximum [size](https://laravel.com/docs/6.x/validation#rule-size) in kilobytes, you can filter out which [mime types](https://laravel.com/docs/6.x/validation#rule-mimetypes) or [file extensions](https://laravel.com/docs/6.x/validation#rule-mimes) are accepted.
<!--more-->

If we want to user to upload ZIP file we can combine rules and use something like this:
``` php
return [
	'file' => 'required|file|mimes:zip|size:3072'
];
```

But unlike image, text or pdf file, ZIP is an archive file and can hold multiple different files in it. Let's say we want user to upload a ZIP file but with specific required files and folder structure. As a real life example, if you ever used Wordpress and installed theme for it using ZIP file, Wordpress requires theme ZIP file to have specific files like `style.css` or `post.php`. How can we validate uploaded ZIP file in Laravel to see if it contains all the required files?

## PHP's `libzip` support
PHP has native ZIP file handling support with [libzip](https://libzip.org/) library, to use it, zip extension must to be enabled. ZIP support allows creating ZIP files, reading or editing existing ZIP files with PHP. It has many methods and options, you can find more information in [official documentation page](https://www.php.net/manual/en/book.zip.php).

## Working with ZIP file
To read existing ZIP file's content, first we need to instantiate `ZipArchive` class
``` php
$zip = new \ZipArchive();
```
Then we can use `open()` method and pass ZIP file's absolute path to open the file.
``` php
$zip->open('/absolute/path/to/file.zip');
```
[Method's official documentation](https://www.php.net/manual/en/ziparchive.open.php) states that it returns `true` if ZIP file opened successfully, otherwise integer error code. We can check everything like this:
``` php
$zipStatus = $zip->open('/absolute/path/to/file.zip');
if (zipStatus !== true) {
	threw new \Exception('Could not open ZIP file. Error code: ' . zipStatus);
}

// all good, continue…
```

`ZipArchive` class has several methods to help us read ZIP file's contents, some of them are:
- [`count()`](https://www.php.net/manual/en/ziparchive.count.php) - returns number of files/entities inside ZIP
- [`getFromIndex()`](https://www.php.net/manual/en/ziparchive.getfromindex.php) - returns entity from given index
- [`getNameIndex()`](https://www.php.net/manual/en/ziparchive.getnameindex.php) - returns entity's name from given index
- [`statIndex()`](https://www.php.net/manual/en/ziparchive.statindex.php) - returns entity's details (name, size, etc) from given index.

Having these methods, if we want to get list of available files inside ZIP file. we can do it like this:
``` php
$filesInside = [];

for ($i = 0; $i < $zip->count(); $i++) {
	array_push($filesInside, $zip->getNameIndex($i));
}

$zip->close();
```  
Since one instance of `ZipArchive` can work with one ZIP file at the same time, we need to use `close()` method when we are done with the file. It is especially important when you are looping through multiple ZIP files or creating new ZIP file using this class.

When working with `ZipArchive`, one other thing you need keep in mind that “entity” does not mean files only, it also means all folders and subfolders.
Lets say we have a ZIP file with following content:
```
- invoice.pdf
- profile_picture.jpg
- documents/
  - homework.doc
- bills/
  - january/
    - payment.pdf
```
Using `count()` method on this file will return 7, even though we only have 4 files in it. If we run above code with this file `$filesInside` variable will result to this:
``` php
$files = [
	'invoice.pdf',
	'profile_picture.jpg',
	'documents/',
	'documents/homework.doc',
	'bills/',
	'bills/january/',
	'bills/january/payment.pdf'
];
```
As you can see, not only files but folders also counted as entity and when `getNameIndex()` method used, it returns relative full path for files inside folders.

## Integrating with Laravel
Now that we know how we can get list of files with PHP's ZIP support, let's integrate everything into Laravel and validate if uploaded ZIP file contains  required files.

Imagine we need user to upload a ZIP file, but that file must contain `thumbnail.jpg` file and `style.css` file inside `assets` folder.
```
- thumbnail.jpg
- assets/
  - style.css
```

Here's our controller which handles file upload with `zip_file` form name and has  `REQUIRED_FILES` defined:
``` php
namespace App\Controllers;

use Illuminate\Http\Request;

class UploadController
{
	const REQUIRED_FILES = [
		'thumbnail.jpg',
		'assets/style.css',
	];

	public function upload(Request $request)
	{
		$zip = new \ZipArchive();
		$file = $request->file('zip_file');
	}
}
```
When you retrieve files from `Illuminate\Http\Request`, each file returned is instance of `Illuminate\Http\UploadedFile` . We can use `path()` method on this instance to return absolute path to temporary uploaded file.

Here's how we can open ZIP file and list files inside it with `ZipArchive` class.
``` php
public function upload(Request $request)
{
	$zip = new \ZipArchive();
	$file = $request->file('zip_file');
	$zip->open($file->path());

	$filesInside = [];
	for ($i = 0; $i < $zip->count(); $i++) {
		array_push($filesInside, $zip->getNameIndex($i));
	}
}
```

Now we can use something like `array_intersect()` to compare `REQUIRED_FILES` with `$filesInside`. If intersection element count is not equal to  `REQUIRED_FILES` element count, it means not all required files exist in uploaded ZIP file and we can abort request execution or return validation error.
``` php
$intersection = array_intersect(self::REQUIRED_FILES, $filesInside);

if (count($intersection) !== count(self::REQUIRED_FILES)) {
	abort(422);
}
```

Here's our controller with everything in place:
``` php
namespace App\Controllers;

use Illuminate\Http\Request;

class UploadController
{
	const REQUIRED_FILES = [
		'thumbnail.jpg',
		'assets/style.css',
	];

	public function upload(Request $request)
	{
		$zip = new \ZipArchive();
		$file = $request->file('zip_file');
		$zip->open($file->path());

		$filesInside = [];
		for ($i = 0; $i < $zip->count(); $i++) {
			array_push($filesInside, $zip->getNameIndex($i));
		}

		$intersection = array_intersect(self::REQUIRED_FILES, $filesInside);

		if (count($intersection) !== count(self:: REQUIRED_FILES)) {
			abort(422);
		}

		// ZIP contains all required files, continue
	}
}
```

## Practical use cases
In Laravel applications it is good practice to move validation related stuff outside of controllers. Common options are:
- creating custom [validation rule class](https://laravel.com/docs/6.x/validation#using-rule-objects)
- using [closure validation](https://laravel.com/docs/6.x/validation#using-closures) inside existing request object
- creating [validation extension](https://laravel.com/docs/6.x/validation#using-extensions)

For ZIP content validation purposes I created dedicated Laravel package which does almost the same thing as I explained in this blog post, with little bit more options and functionality. You can check out [orkhanahmadov/laravel-zip-validator](https://github.com/orkhanahmadov/laravel-zip-validator) GitHub page for more information and how to use it.