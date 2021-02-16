---
layout: post
title: Using whereLike to search in multiple columns with Laravel
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, eloquent]
comments: false
---

For a project I’m working on, I find myself performing a search query against multiple Eloquent models, where I search for a **term** in <ins>multiple</ins> and <ins>different</ins> attributes within each model.

As far as can see, there are 2 ways to tackle this.

The first would be to extend the *Eloquent Query Builder* using `macros`. macros will let you add the search feature to **all** your models. If this sounds like what you looking for, then [Freek Van der Herten](https://twitter.com/freekmurze) got your back, Freek has a great [blog post](https://freek.dev/1182-searching-models-using-a-where-like-query-in-laravel) explaining this approach. As a matter of fact, this blog post is highly based on his approach.

The second one, is to use *Local Query Scope*. Local scopes allow you to define **common** sets of **query constraints** that you may easily **re-use** throughout your application. And to make it more *re-usable* I'm wrapping the **search scope** inside a `trait` so it's easy to share this feature within different models.

In this post, I will share with you my experience and the approach I chose to improve the readability and the reusability of my search functionality through a `trait` I named `Searchable`.

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
User::search('searchTerm', ['column1', 'column2'])->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `column1` LIKE '%searchTerm%' 
 *     OR `column2` LIKE '%searchTerm%'
 */
``` 

The order of the arguments is not really important, as long as you respect some conventions, of which, the *searchTerm* is always a `string`, and the *attributes* is an `array`.  
With that in mind, the above snippet could easily be written like below and still got the same results:

#### snippet 2
```php
User::search(['column1', 'column2'], 'searchTerm')->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `column1` LIKE '%searchTerm%' 
 *     OR `column2` LIKE '%searchTerm%'
 */
``` 

Sometimes you may only want to pass *searchTerm* as the only argument, that's fine too. As long as your model has, either a public `$searchable` property or a public `searchable` method returning an `array` of *attributes* you wish to perform the search against, everything is going to be fine.

#### snippet 3
```php
class User extends Model
{
    /**
     * Searchable attributes
     *
     * @return string[]
     */
    public $searchable = ['column1', 'column2'];

    // OR

    /**
     * Searchable attributes
     *
     * @return string[]
     */
    public static function searchable()
    {
        return ['column1', 'column2'];
    }
}

//...

User::search('searchTerm')->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `column1` LIKE '%searchTerm%' 
 *     OR `column2` LIKE '%searchTerm%'
 */
``` 

Or maybe you don't want to pass any arguments at all.

#### snippet 4
```php
User::search()->get();

/**
 * SELECT * FROM `users` 
 *     WHERE `column1` LIKE '%searchTerm%' 
 *     OR `column2` LIKE '%searchTerm%'
 */
``` 

The last snippet works, because behind the scenes, we check if a specific query parameter is present in the `Request`, if it's the case we use it the get the *searchTerm*. And for the *attributes* we get them from `$searchable` property or `searchable` method.

By this far, I hope this opens your appetite to follow along with me to implement this feature.

I will start by creating `Searchable` trait, I usually keep the model's traits inside `App\Concerns\Models` namespace, but this just a personal preference, you can put it wherever you think suits your project the best.

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