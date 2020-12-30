---
layout: post
title: Building multilingual showroom gallery with laravel - part 2
subtitle: Database Modeling
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, database]
comments: true
---

Before getting our hands dirty with database migrations, let's take a moment and think about what kind of tables we need to achieve our goal. Diving in database modeling techniques and advanced database design falls outside the scope of this article, but, here’s some basic business rules to get us start with our modelization:

_Website is multilingual, Enlish and French as languages of choise._  
_Product belongs to one or many categories._  
_Categories could have many sub categories (deep to one level)._  
_Category could have no parent or one parent at max._

I always prefer to define the most important entities just to get the ball rolling. The given rules above, could easly accomplished with these tables:

* Artisans
* Categories
* Products

But, since this is a multilingual website, we gon to need other tables to store localized fields. For that I plan to use 
[mcamara/laravel-localization](https://github.com/mcamara/laravel-localization) package. This package offers the following:

* Detect language from browser
* Smart redirects (Save locale in session/cookie)
* Smart routing (Define your routes only once, no matter how many languages you use)
* Translatable Routes
* Supports caching & testing
* Option to hide default locale in url
* Many snippets and helpers (like language selector)

With that in mind, we end up with these tables:

* Artisans
* Artisan Translations
* Categories
* Category Translations
* Products
* Product Translations
* Product Categories

I think this is enough to get the application up and running, other tables may be added along the way as needed, but let's keep it simple for the time been.

Notice that there is new table named _"Prodcut Categories"_, that's the result of the `many-to-many` relationship between _"Product"_ table and _"Categories"_ table, more on that later.


## Migrations

I find it easy to create migrations using the `php artisan make:model` command, because a table needs to be represented by an Eloquent model anyway, and I can also create a controller and a factory for the model by using the `--controller` and `--factory` options.  
For more info display the help with one of those commands `php artisan help make:model` or `php artisan make:model --help`


### Artisan

```bash
php artisan make:model Artisan --migration
```
---

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateArtisansTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('artisans', function (Blueprint $table) {
            $table->id();
            $table->string('first_name');
            $table->string('last_name');
            $table->string('email')->unique();
            $table->char('gender', 1)->nullable();
            $table->boolean('active')->default(true);
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('artisans');
    }
}
```

To keep the migrations as simple as possible I kept the fields at the bottom minimum , the migration it self is self-explanitory, the only thing I want to point out here is for _gender_ field, I intent to store it in char type with 1 character lenght, so I can use `m` for male, and `f` for female.

Next thing is "Artisan Translations" table, this table contains one localized field, it's _bio_.

Exactly like the first time, we are creating `ArtisanTranslations` model with the migration.

```bash
php artisan make:model ArtisanTranslations --migration
```
---

```php
public function up()
{
    Schema::create('artisan_translations', function (Blueprint $table) {
        $table->id();
        $table->foreignId('artisan_id')->constrained()->cascadeOnDelete();
        $table->text('bio')->nullable();
        $table->char('locale', 2)->index();
        $table->unique(['artisan_id','locale']);
    });
}
```