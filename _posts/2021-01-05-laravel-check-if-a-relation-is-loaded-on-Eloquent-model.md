---
layout: post
title: Laravel check if a relation is loaded on Eloquent model
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, eloquent]
comments: false
---

## TL;DR

```php
$model->relationLoaded('myRelation');
```

#### Usage

```php
if ($model->relationLoaded('myRelation')) {
    // Look! "myRelation" is already loaded.
}
```

#### How it works

This Eloquent method, checks if the key _"myRelation"_ exists in the relations array of the Eloquent model, if it exists, wich means the realtionships is already been loaded, it returns `true`, otherwise, it returns `false`.

#### Function signature

```php
/**
 * Determine if the given relation is loaded.
 *
 * @param  string  $key
 * @return bool
 */
public function relationLoaded($key)
{
    return array_key_exists($key, $this->relations);
}
```

## How this method benefited me

Briefly, in one of my projects, I had two tables _"artisans"_, _"products"_ where artisans `hasMany` products, and product `belongs` to a an artisan, as you can see below.

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Artisan extends Model
{
    //...

    /**
     * Products that belongs the artisan
     *
     * @return \Illuminate\Database\Eloquent\Relations\HasMany
     */
    public function products()
    {
        return $this->hasMany(Product::class);
    }

    //...
}
```

---

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    //...

    /**
     * Product's owner
     *
     * @return \Illuminate\Database\Eloquent\Relations\BelongsTo
     */
    public function artisan()
    {
        return $this->belongsTo(Artisan::class);
    }

    //...
}
```

Now whenever I want to fetch an artisan with his products I do this:

```php
$artisan = Artisan::with(['products'])->get();
```

In the project, the `Artisan` model has more relationships, but for simplicity purposes, I will only keep products relationship.

Later on, I frequently find my self eager loading only artisan's active produtcs. I got to accomplish that by passing an array of relationships to the `with` method where the array key is a relationship name (in our case _products_) and the array value is a closure that adds additional constraints to the eager loading query:

```php
$artisan = Artisan::with(['products' => function ($query) {
    $query->where('active', 1);
}])->get();
```

So, I thought it may be a good idea to create `activeProducts` relationship in the `Artisan` model to make it more simple and clear. So I did.

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Artisan extends Model
{
    //...

    /**
     * Products that belongs to the artisan
     *
     * @return \Illuminate\Database\Eloquent\Relations\HasMany
     */
    public function products()
    {
        return $this->hasMany(Product::class);
    }

    /**
     * "Active" products that belong to the artisan.
     *
     * @return \Illuminate\Database\Eloquent\Relations\HasMany
     */
    public function activeProducts()
    {
        return $this->products()->where('active', 1);
    }

    //...
}
```

Now, without the need to pass a closure to `products` relationship to fetch artisan's active products, i can accomplish the same thing with just this:

 ```php
$artisan = Artisan::with(['activeProducts'])->get();
```

(wip)

Although this brings simplicity, but, it raises a problem

sub views to show products

you need to know wich relationship is loaded, so you don't need to unneccery hit the database.

relationLoaded to the rescue.

now with this i can use the same sub view for both where model is loaded with `products`  or `activeProducts`