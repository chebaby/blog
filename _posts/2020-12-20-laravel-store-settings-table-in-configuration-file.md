---
layout: post
title: Store settings table in a configuration file
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel]
comments: false
---

If you find yourself in a situation where you need to store settings table in a configuration file, but in the same time you don't want to sacrifice given the User/Admin the ability to update those settings when needed, then you've come to the right place.

One of the reasons you may want to store your application settings in file is avoiding hitting the database too much.  
Well, I faced this problem myself in one of the projects I worked on, it was a relatively a small project, storing my settings in a configuration file may not gain me too much in perfomance, but for my own peace of mind, I did it anyway.

Here is schema defination of settings table:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateSettingsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('settings', function (Blueprint $table) {
            $table->id();
            $table->string('key')->unique();
            $table->text('value');
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
        Schema::dropIfExists('settings');
    }
}
```

As you can see, it's `key => value` combination.

Well the solution turns out to be pretty simple, all I had to do was taking adavantage of the [Eloquent Model Events](https://laravel.com/docs/8.x/eloquent#events), to be more precise I used `saved` event, because the `saving`/`saved` events will dispatch when a model is <u>created</u> or <u>updated</u>.

>This is tested in Laravel 8 application, but i believe it will work fine from Laravel version 5.5 and up.

```php
<?php

namespace App\Models;

use Illuminate\Support\Facades\File;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Setting extends Model
{
    use HasFactory;

    /**
     * The attributes that aren't mass assignable.
     *
     * @var array
     */
    protected $guarded = [];

    /**
     * The "booted" method of the model.
     *
     * @return void
     */
    protected static function booted()
    {
        static::saved(function () {

            $settings           = static::pluck('value', 'key')->toArray();

            $parsable_string    = var_export($settings, true);

            $content            = "<?php return {$parsable_string};";

            File::put(config_path('app_settings.php'), $content);
        });
    }

    //...
}
```

Nothing complexe really goin on here, first, through out the Eloquent `pluck` method we retrieve an `Illuminate\Support\Collection` instance containing the keys/values.  
Chaining `toArray` method converts the collection into a plain PHP `array`.

PHP function `var_export`returns a parsable string representation of the `$settings` variable.

I also use the `config_path` function to generate a _fully qualified path_ to a file within the application's configuration directory, a file I named **app_settings.php**. You are free to change it to whatever suits you best.  
The `put` method is used to store file contents on the disk.


I did one more thing, I added `/config/app_settings.php` to `.gitignore` file.