---
layout: post
title: Laravel Package Development - Create Searchable Package
share-description: I will show you how I built a simple laravel package I named laravel-searchable. laravel-searchable is a trait for Laravel 6+ that adds a simple search function to Eloquent Models. laravel-searchable 
 make it simple for you to perform searches in multiple columns.
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, package]
comments: false
head-extra: article_structured_data.html
---

## Setup

In the [previous](https://echebaby.com/blog/2021-02-13-searchable-trait-to-search-in-multiple-columns-with-laravel/) article, we have seen how we can create a searchable trait that adds searching functionality to our Eloquent Models.
Today we are going to encapsulate this trait in a package so it can be used in different projects easily.

If you have followed along the previous article, you will know that we had two mandatory files:

`app/Concerns/Models/Searchable` trait which contains the logic of the search query scope.  
`config/searchable.php` configuration file which contains default configuration options, in our case it’s just a request query key.

An easy way to quickly get started with Laravel Package Developing is using [Laravel Package Boilerplate](https://laravelpackageboilerplate.com/#/){:target="_blank"} it comes with pre-configured continuous integration services out of the box.

### Laravel Package Boilerplate

Since we are building specifically a Laravel package, select Laravel and click next button.

<img src="{{ 'assets/img/laravel-searchable-package/laravelpackageboilerplate_first_step.png' | relative_url }}" alt="Laravel Package Boilerplate - first step" />

Fill in some basic informations about the package, and click next button.

<img src="{{ 'assets/img/laravel-searchable-package/laravelpackageboilerplate_second_step.png' | relative_url }}" alt="Laravel Package Boilerplate - second step" />

Finally download the Zip file.

<img src="{{ 'assets/img/laravel-searchable-package/laravelpackageboilerplate_third_step.png' | relative_url }}" alt="Laravel Package Boilerplate - third step" />

Extract the Zip to a folder, and open it in your preferred IDE.

<img src="{{ 'assets/img/laravel-searchable-package/laravel-searchable-folder-structure.png' | relative_url }}" alt="Laravel Package Boilerplate - folder structure" />

Inside the `src` folder is where we put our Searchable trait. But before we do that there are some updates I have to make to the default boilerplate files.

One of these is to remove `LaravelSearchableFacade` we don’t need that.

We remove it also from the `extra` section of the package's `composer.json` file.

```json
"extra": {
   "laravel": {
       "providers": [
           "Chebaby\\LaravelSearchable\\LaravelSearchableServiceProvider"
       ],
       "aliases": {
           "LaravelSearchable": "Chebaby\\LaravelSearchable\\LaravelSearchableFacade"
       }
   }
}

```

Remove `aliases` key.

```json
"extra": {
   "laravel": {
       "providers": [
           "Chebaby\\LaravelSearchable\\LaravelSearchableServiceProvider"
       ]
   }
}

```

And then remove it's reference from `register` method of `LaravelSearchableServiceProvider`.


One more thing is to rename the `LaravelSearchable` *class* to `Searchable` *trait*.

## Development

Inside the config *folder* there is a config *file* where we can place custom package configurations. As stated previously, for now, it’s just a request query key.

```php
<?php

return [
   // query key
   // e.g http://example.com/search?q=searchTerm
   'key' => 'q',
];

```

For the Searchable logic we can copy it from tha last section in the previous [article](https://echebaby.com/blog/2021-02-13-searchable-trait-to-search-in-multiple-columns-with-laravel/#final-results) to the `src/Searchable.php` file of our package:

```php
<?php

namespace App\Concerns\Models;

use Illuminate\Support\Arr;
use Illuminate\Database\Eloquent\Builder;

trait Searchable
{
    /**
     * Scope a query to search for a term in the attributes
     *
     * @param \Illuminate\Database\Eloquent\Builder $query
     * @return \Illuminate\Database\Eloquent\Builder
     */
    protected function scopeSearch($query)
    {
        [$searchTerm, $attributes] = $this->parseArguments(func_get_args());

        if (!$searchTerm || !$attributes) {
            return $query;
        }

        return $query->where(function (Builder $query) use ($attributes, $searchTerm) {
            foreach (Arr::wrap($attributes) as $attribute) {
                $query->when(
                    str_contains($attribute, '.'),
                    function (Builder $query) use ($attribute, $searchTerm) {
                        [$relationName, $relationAttribute] = explode('.', $attribute);

                        $query->orWhereHas($relationName, function (Builder $query) use ($relationAttribute, $searchTerm) {
                            $query->where($relationAttribute, 'LIKE', "%{$searchTerm}%");
                        });
                    },
                    function (Builder $query) use ($attribute, $searchTerm) {
                        $query->orWhere($attribute, 'LIKE', "%{$searchTerm}%");
                    }
                );
            }
        });
    }

    /**
     * Parse search scope arguments
     *
     * @param array $arguments
     * @return array
     */
    private function parseArguments(array $arguments)
    {
        $args_count = count($arguments);

        switch ($args_count) {
            case 1:
                return [request(config('searchable.key')), $this->searchableAttributes()];
                break;

            case 2:
                return is_string($arguments[1])
                    ? [$arguments[1], $this->searchableAttributes()]
                    : [request(config('searchable.key')), $arguments[1]];
                break;

            case 3:
                return is_string($arguments[1])
                    ? [$arguments[1], $arguments[2]]
                    : [$arguments[2], $arguments[1]];
                break;

            default:
                return [null, []];
                break;
        }
    }

    /**
     * Get searchable columns
     *
     * @return array
     */
    public function searchableAttributes()
    {
        if (method_exists($this, 'searchable')) {
            return $this->searchable();
        }

        return property_exists($this, 'searchable') ? $this->searchable : [];
    }
}

```

## Tests

When writing packages, your package will not typically have access to all of Laravel's testing helpers. If you would like to be able to write your package tests as if the package were installed inside a typical Laravel application, you may use the [Orchestral Testbench](https://github.com/orchestral/testbench) package. Which is already in the `composer.json` of the Laravel Package Boilerplate, so all we have to do is to run `composer install` to install package dependencies.

### Memory SQLite Connection

To reduce setup configuration, you could use testing database connection (`:memory:` with sqlite driver) via setting it up under `getEnvironmentSetUp()` or by defining it under PHPUnit Configuration File `phpunit.xml.dist`:

```html
<phpunit>

    // ...

    <php>
        <env name="DB_CONNECTION" value="testing"/>
    </php>

</phpunit>

```

### Running Testing Migrations

To run migrations that are only used for testing purposes and not part of your package, add the following to your base test class `tests/TestCase.php`

```php
<?php

namespace Chebaby\LaravelSearchable\Tests;

use Illuminate\Foundation\Application;
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Orchestra\Testbench\TestCase as BaseTestCase;

class TestCase extends BaseTestCase
{
    protected function setUp(): void
    {
        parent::setUp();

        $this->setUpDatabase($this->app);
    }

    protected function setUpDatabase(Application $app)
    {
        Schema::create('countries', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name')->nullable();
        });

        Schema::create('users', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name')->nullable();
            $table->string('email')->nullable();
            $table->string('phone')->nullable();
            $table->unsignedBigInteger('country_id')->nullable();
            $table->foreign('country_id')->references('id')->on('countries')->onDelete('SET NULL');
        });
    }
}

```

We create migration to two table `countries` and `users` so we can test searching for an attribute in a related model.

### Create User/Country Models

Inside the `tests` folder we create another folder with the name **Models**, you guessed it right, it is where our models live.

<img src="{{ 'assets/img/laravel-searchable-package/laravel-searchable-tests-user-model.png' | relative_url }}" alt="Laravel Searchable Package - testing models" />

Rename ExampleTest to SearchableTest to start writing our tests.

In the last [article](https://echebaby.com/blog/2021-02-13-searchable-trait-to-search-in-multiple-columns-with-laravel/), there is some code snippets showing the trait usage, based on those snippets, I'm whriting tests to cover those use cases.

#### [Snippet 1](https://echebaby.com/blog/2021-02-13-searchable-trait-to-search-in-multiple-columns-with-laravel/#snippet-1)

```php
User::search('searchTerm', ['name', 'email'])->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 */
``` 

#### Testing snippet 1

```php
/** @test */
public function it_can_search_for_a_term_in_multiple_attributes()
{
    User::create([
        'name' => 'john doe',
        'email' => 'john@doe.com'
    ]);

    $user = User::search('john', ['name', 'email'])->first();

    $this->assertEquals('john doe', $user->name);
    $this->assertEquals('john@doe.com', $user->email);
}
```

#### Testing snippet 1 - result

<img src="{{ 'assets/img/laravel-searchable-package/laravel-searchable-tests-snippet-1.png' | relative_url }}" alt="Laravel Searchable Package - testing snippet 1" />

#### [Snippet 2](https://echebaby.com/blog/2021-02-13-searchable-trait-to-search-in-multiple-columns-with-laravel/#snippet-2)

```php
User::search(['name', 'email'], 'searchTerm')->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 */
```

#### Testing snippet 2

```php
/** @test */
public function it_can_search_in_multiple_attributes_for_a_term()
{
    User::create([
        'name' => 'jane doe',
        'email' => 'jane@doe.com'
    ]);

    $user = User::search(['name', 'email'], 'jane')->first();

    $this->assertEquals('jane doe', $user->name);
    $this->assertEquals('jane@doe.com', $user->email);
}
```

#### Testing snippet 2 - result

<img src="{{ 'assets/img/laravel-searchable-package/laravel-searchable-tests-snippet-2.png' | relative_url }}" alt="Laravel Searchable Package - testing snippet 2" />

#### [Snippet 3](https://echebaby.com/blog/2021-02-13-searchable-trait-to-search-in-multiple-columns-with-laravel/#snippet-3)

```php
User::search('searchTerm')->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 */
```

#### Testing snippet 3

```php
/** @test */
public function it_can_search_for_a_term_without_passing_attributes_using_searchable_property()
{
    UserWithSearchableProperty::create([
        'name'  => 'Johnny doe',
        'email' => 'Johnny@doe.com'
    ]);

    $user = UserWithSearchableProperty::search('johnny')->first();

    $this->assertEquals('Johnny doe', $user->name);
    $this->assertEquals('Johnny@doe.com', $user->email);
}

/** @test */
public function it_can_search_for_a_term_without_passing_attributes_using_searchable_method()
{
    UserWithSearchableMethod::create([
        'name'  => 'Richard Roe',
        'email' => 'richard@doe.com'
    ]);

    $user = UserWithSearchableMethod::search('richard')->first();

    $this->assertEquals('Richard Roe', $user->name);
    $this->assertEquals('richard@doe.com', $user->email);
}
```

#### Testing snippet 3 - result

<img src="{{ 'assets/img/laravel-searchable-package/laravel-searchable-tests-snippet-3.png' | relative_url }}" alt="Laravel Searchable Package - testing snippet 3" />

#### [Snippet 4](https://echebaby.com/blog/2021-02-13-searchable-trait-to-search-in-multiple-columns-with-laravel/#snippet-4)

```php
User::search(['name', 'email', 'phone'])->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 *     OR `phone` LIKE '%searchTerm%'
 */
```

#### Testing snippet 4

```php
/** @test */
public function it_can_search_in_multiple_attributes_without_passing_a_term()
{
    User::create([
        'name'  => 'Janie Roe',
        'email' => 'janie@doe.com',
    ]);

    // simulate GET request with query parameter "?q=janie"
    request()->query->add(['q' => 'janie']);

    $user = User::search(['name', 'email', 'phone'])->first();

    $this->assertEquals('Janie Roe', $user->name);
    $this->assertEquals('janie@doe.com', $user->email);

    request()->query->remove('q');

    // update request query key
    config(['searchable.key' => 'keyword']);

    // simulate GET request with query parameter "?keyword=janie"
    request()->query->add(['keyword' => 'janie']);

    $user = User::search(['name', 'email', 'phone'])->first();

    $this->assertEquals('Janie Roe', $user->name);
    $this->assertEquals('janie@doe.com', $user->email);
}
```

#### Testing snippet 4 - result

<img src="{{ 'assets/img/laravel-searchable-package/laravel-searchable-tests-snippet-4.png' | relative_url }}" alt="Laravel Searchable Package - testing snippet 4" />