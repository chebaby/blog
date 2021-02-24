---
layout: post
title: Using whereLike to search in multiple columns with Laravel
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, eloquent]
comments: false
---

For a project I’m working on, I find myself performing a search query against multiple Eloquent models, where I search for a **term** in <ins>multiple</ins> and <ins>different</ins> attributes within each model.

Possibilities to solve this are several, I'm sure, but as far as can see, there are 2 ways to tackle this.

The first would be to extend the *Eloquent Query Builder* using `macros`. macros will let you add the search feature to **ALL** your models. If this sounds like what you looking for, then [Freek Van der Herten](https://twitter.com/freekmurze) got your back, Freek has a great [blog post](https://freek.dev/1182-searching-models-using-a-where-like-query-in-laravel) explaining this approach. As a matter of fact, this blog post is highly based on his approach.

The second one, is to use *Local Query Scope*. Local scopes allow you to define **common** sets of **query constraints** that you may easily **re-use** throughout your application. And to make it even more *re-usable* I'm wrapping the **search scope** inside a `trait` so it's easy to apply to every model we want to make searchable.

In this post, I will share with you my experience and the approach I choose to improve the readability and the reusability of my search functionality through a `trait` I named `Searchable`.

But before getting to the "real stuff", here is a quick peek to the end result.

## Usage

```php
class User extends Model
{
    use Searchable;
}
``` 

#### snippet 1
```php
User::search('searchTerm', ['name', 'email'])->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 */
``` 

The order of the arguments is not really important, as long as you respect some conventions, of which, the *searchTerm* is always a `string`, and the *attributes* is an `array`. With that in mind, the above snippet could easily be written like below and still got the same results:

#### snippet 2
```php
User::search(['name', 'email'], 'searchTerm')->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 */
``` 

Sometimes you may only want to pass *searchTerm* as the only argument, that's fine too. As long as your model has, either a public `$searchable` property or a public `searchable` method returning an `array` of *attributes* you wish to perform the search against, everything is going to be fine.

#### snippet 3
```php
class User extends Model
{
    use Searchable;

    /**
     * Searchable attributes
     *
     * @return string[]
     */
    public $searchable = ['name', 'email'];

    // OR

    /**
     * Searchable attributes
     *
     * @return string[]
     */
    public static function searchable()
    {
        return ['name', 'email'];
    }
}

//...

User::search('searchTerm')->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 */
``` 

Or maybe you only want to pass *attributes*, and let the "magic" guess for the *searchTerm*, as I'm going to explain in a second, that's possible too.

#### snippet 4
```php
User::search(['name', 'email', 'phone'])->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 *     OR `phone` LIKE '%searchTerm%'
 */
``` 

The last snippets works, because behind the scenes, I check if a specific query parameter key is present in the `Request`, if it's the case I use it the get the *searchTerm*.

What about the case where you don't want to pass any arguments at all. Well, that works too.

#### snippet 5
```php
User::search()->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `name` LIKE '%searchTerm%' 
 *     OR `email` LIKE '%searchTerm%'
 */
``` 


### Support for relations

You can also search for attributes in related models.

#### snippet 6
```php
User::search('searchTerm', ['name', 'country.name'])->get();

/**
 * SELECT * FROM `users` 
 * WHERE (
 *    `name` LIKE '%searchTerm%' OR 
 *    EXISTS (
 *        SELECT * FROM `countries` WHERE `users`.`country_id` = `countries`.`id` 
 *        AND `name` LIKE '%searchTerm%'
 *    )
 *)
 */
``` 

As you can conclude from the generated SQL, we are looking for users where name <ins>contains</ins> *searchTerm* `OR` users where country name <ins>contains</ins> *searchTerm*. 

By this far, I hope this opens your appetite to follow along with me to implement this feature.

## Implementation

I will start by creating `Searchable` trait, I usually keep the model's traits in the `App\Concerns\Models` namespace, but this just a personal preference, you can put it wherever you think suits your project the best.

```php
<?php

namespace App\Concerns\Models;

trait Searchable
{
    /**
     * Scope a query to search for term in the attributes
     *
     * @param \Illuminate\Database\Eloquent\Builder $query
     * @return \Illuminate\Database\Eloquent\Builder
     */
    protected function scopeSearch($query)
    {
        // search logic...
    }
}
```

By now you know that there is two **optional** arguments we can pass to search scope, a *$searchTerm* `string` and an `array` of *$attributes*.  
While `$query` argument is always present as it's passed by *Laravel Query Builder*.

It seems to me that I should first parse search scope arguments, for that reason I created `parseArguments` method to helps us out with the parsing.

```php
<?php

namespace App\Concerns\Models;

trait Searchable
{
    /**
     * Scope a query to search for term in the attributes
     *
     * @param \Illuminate\Database\Eloquent\Builder $query
     * @return \Illuminate\Database\Eloquent\Builder
     */
    protected function scopeSearch($query)
    {
        [$searchTerm, $attributes] = $this->parseArguments(func_get_args());
    }

    /**
     * Parse search scope arguments
     *
     * @param array $arguments
     * @return array
     */
    private function parseArguments(array $arguments)
    {
        // parsing...
    }
}
```

`func_get_args` gets an array of the function's argument list. In our case it's a list of argument passed to *scopeSearch* which are *$query* `Builder`, *$searchTerm* `string` and *$attributes* `array`.

Here is the entire `parseArguments` definition alongside the `searchableAttributes` method definition.

**N.B.** `parseArguments` returns an `array` where the *$searchTerm* is at index `0` and *$attrubutes* at index `1`.

```php
<?php
    //...
    
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
```

`searchableAttributes` main job is to get searchable attributes array difined in the model *(e.g: User)*. It's used in the case where the **$attributes** argument is not provided in search scope *(see snippet [3](#snippet-3) and [5](#snippet-5))*. `searchableAttributes` first looks for a method in the model named *searchable* if it exist, if so, it returns the result, if not, it looks for a property named *$searchable*, again if it exist it returns it's content, if not, an empty array is returned instead.

For `parseArguments` first we count how many arguments passed to the search scope. The number of arguments is important here as it will tell us how our scope is been used *(see the [usage](#usage) section)*.

One thing to note is that the value of `$args_count` will never be `0` (that's why you won't find a `case` statement for `0` in the `switch`) because *$query* `Builder` is always passed by Laravel as the first argument, which also means, it is the argument at index 0 of `$arguments` array.

Ok, let's break it down case by case.

#### case 1

```php
case 1:
    return [request(config('searchable.key')), $this->searchableAttributes()];
    break;
```

Correspond to [snippet 5](#snippet-5). In this case, no arguments are passed to search scope, so it's up to us get the *searchTerm* from the `Request` and the *attributes* from the `Model`.

Let's take a closer look at `request(config('searchable.key'))`. As a Laravel user you know, as the name implies, that [config](https://laravel.com/docs/8.x/helpers#method-config) access a value in a config file using the "dot" syntax, which includes the name of the file and the option you wish to access.

So, to get the query parameter key that presumably holds your *searchTerm* value, you need to have configuration file (`searchable.php`) in `config` directory.

```php
return [
    // query parameter key
    // e.g http://example.com/search?q=searchTerm
    'key' => 'q',
];
```

In this example, the query parameter key that will be used to get *searchTerm* value is **q**, so as long as you `Request` has <ins>filled</ins> **q** parameter you're good to go.


#### case 2

```php
case 2:
    return is_string($arguments[1])
        ? [$arguments[1], $this->searchableAttributes()]
        : [request(config('searchable.key')), $arguments[1]];
    break;
```

Correspond to [snippet 3](#snippet-3) *OR* [snippet 4](#snippet-4). In this case, either *$searchTerm* `string` *OR* *$attributes* `array` alongside *$query* `Builder` are passed to the search scope. this convention of string/array make easy for us to distinguich what type of argument is passed to the search scope.

So, if it's the *$searchTerm* `string` we get the *$attributes* `array` by involking searchableAttrbutes method, else it's the *$attributes* `array` we get the *$searchTerm* from `Request` using configured query parameter key.


#### case 3

```php
case 3:
    return is_string($arguments[1])
        ? [$arguments[1], $arguments[2]]
        : [$arguments[2], $arguments[1]];
    break;
```

Correspond to [snippet 1](#snippet-1) *OR* [snippet 2](#snippet-2) *OR* [snippet 6](#snippet-6). both *$searchTerm* **and**
*$attributes* are present as arguements to our search scope, regardless of theire order.  
We make sure the order of arguments in the returned array is as expected: `[string $searchTerm, array $attributes]`.

After argument parsing, the logical next step is to build the query, and it's exaclty what we are going to do next.

### Build the query


**To be continued...**




