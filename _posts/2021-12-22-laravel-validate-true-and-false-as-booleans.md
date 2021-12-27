---
layout: post
title: Laravel validate true and false as booleans
share-description: I will share with you the approach I take to properly validate a form input with value the string value of 'true' or 'flase'.
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, validation]
comments: false
head-extra: article_structured_data.html
---


Every once in a while you come across a situation where you need to validate a request input as a boolean, and the input value is `'true'` or `'false'` (notice that I wrapped the values inside single quotes to indicate that those are actually strings). <br>
You would expect, *as I did*, that this will just work out of the box and Laravel validator will just see them as booleans. But that is not the case, instead, you will be hit by this beautiful error message `'The inputName field must be true or false.'`

To deeply understand why this is happening, let's go through an example, let's say that we have a form request that we call `PostRequest`.

>Form requests are custom request classes that encapsulate their own validation and authorization logic. <br><br>-- Laravel docs.


```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class PostRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     *
     * @return bool
     */
    public function authorize()
    {
        return true;
    }

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
            'is_published' => 'required|boolean',
        ];
    }
}
```

`PostRequest`, for simplicity purposes, has a single request input that we need to validate as boolean, we will call that `is_published`, and let's say that `is_published` value coming to us from the client is `'true'` or `'false'`. <br>
At this point we already know that validation wouldn't pass, so, how can we handle this? 


## Beforehand

Laravel boolean rule doesn't validate `'true'` and `'false'` as valid booleans, we can actually see that in `validateBoolean()` method located in `Illuminate/Validation/Concerns/ValidatesAttributes`.

```php
<?php

namespace Illuminate\Validation\Concerns;

/**
 * Validate that an attribute is a boolean.
 *
 * @param  string  $attribute
 * @param  mixed  $value
 * @return bool
 */
public function validateBoolean($attribute, $value)
{
    $acceptable = [true, false, 0, 1, '0', '1'];

    return in_array($value, $acceptable, true);
}
```

We can also see that in this tests suite. *(the one in the middle)*

```php
<?php

public function testValidateBoolean()
{
    $trans = $this->getIlluminateArrayTranslator();
    $v = new Validator($trans, ['foo' => 'no'], ['foo' => 'Boolean']);
    $this->assertFalse($v->passes());

    $v = new Validator($trans, ['foo' => 'yes'], ['foo' => 'Boolean']);
    $this->assertFalse($v->passes());
```
```php
    $v = new Validator($trans, ['foo' => 'false'], ['foo' => 'Boolean']);
    $this->assertFalse($v->passes());

    $v = new Validator($trans, ['foo' => 'true'], ['foo' => 'Boolean']);
    $this->assertFalse($v->passes());
```
```php
    $v = new Validator($trans, [], ['foo' => 'Boolean']);
    $this->assertTrue($v->passes());

    $v = new Validator($trans, ['foo' => false], ['foo' => 'Boolean']);
    $this->assertTrue($v->passes());

    $v = new Validator($trans, ['foo' => true], ['foo' => 'Boolean']);
    $this->assertTrue($v->passes());

    $v = new Validator($trans, ['foo' => '1'], ['foo' => 'Boolean']);
    $this->assertTrue($v->passes());

    $v = new Validator($trans, ['foo' => 1], ['foo' => 'Boolean']);
    $this->assertTrue($v->passes());

    $v = new Validator($trans, ['foo' => '0'], ['foo' => 'Boolean']);
    $this->assertTrue($v->passes());

    $v = new Validator($trans, ['foo' => 0], ['foo' => 'Boolean']);
    $this->assertTrue($v->passes());
}
```

## Solutions

### The first approach - use prepareForValidation method

Let's start with what I think it's a more easy approach to implement, it's what Laravel calls "Preparing Input For Validation", 
and that's done by using `prepareForValidation()` method. Like its name reveals, this method allows us to add new request inputs or update existing request inputs before going through validation rules.

So in our small example here, we will try to convert the `is_published` value to an actual boolean, and merge it back to the original request.

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class PostRequest extends FormRequest
{
    // ...

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
            'is_published' => 'required|boolean',
        ];
    }

    /**
     * Prepare inputs for validation.
     *
     * @return void
     */
    protected function prepareForValidation()
    {
        $this->merge([
            'is_published' => $this->toBoolean($this->is_published),
        ]);
    }

    /**
     * Convert to boolean
     *
     * @param $booleable
     * @return boolean
     */
    private function toBoolean($booleable)
    {
        return filter_var($booleable, FILTER_VALIDATE_BOOLEAN);
    }
}
```

`FILTER_VALIDATE_BOOLEAN` tries to be smart, recognizing words like *`'Yes'`*, *`'No'`*, *`'Off'`*, *`'On'`*, *`'true'`* and *`'false'`*, and is not case-sensitive when validating strings.

`FILTER_VALIDATE_BOOLEAN` returns **true** for *`'1'`*, *`'true'`*, *`'on'`* and *`'yes'`*. Returns **false** otherwise.


### The second approach - use a custom validation rule

While the first approach works perfectly, there is a "classy" way to validate the input as boolean, and that's by creating custom validation rules using rule objects.

>Laravel provides a variety of helpful validation rules; however, you may wish to specify some of your own. One method of registering custom validation rules is using rule objects. To generate a new rule object, you may use the make:rule Artisan command. <br><br>-- Laravel docs.

Let's use this command to generate a rule that verifies a string value of `true` and `false` as boolean. 

>Laravel will place the new rule in the `app/Rules` directory. If this directory does not exist, Laravel will create it when you execute the Artisan command to create your rule. <br><br>-- Laravel docs.

```
php artisan make:rule Boolean
```

As promised, a `Boolean` class is created in `app/Rules` namespace, and here is what it looks like

```php
<?php

namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;

class Boolean implements Rule
{
    /**
     * Create a new rule instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Determine if the validation rule passes.
     *
     * @param  string  $attribute
     * @param  mixed  $value
     * @return bool
     */
    public function passes($attribute, $value)
    {
        //
    }

    /**
     * Get the validation error message.
     *
     * @return string
     */
    public function message()
    {
        return 'The validation error message.';
    }
}
```

Once the `Boolean` rule has been created, we are ready to define its behavior. A rule object contains two methods: `passes` and `message`. The passes method receives the attribute *value* and *name*, and should return `true` or `false` depending on whether the attribute value is valid or not. The message method should return the validation error message that should be used when validation fails.

#### Small turn - make global helper functions

But just before doing that, it may be useful to extract `toBoolean` from before to its own function and make it available globally.

An easy and efficient way of creating global functions in Laravel is to autoload it directly from Composer. The autoload section of composer accepts a `files` array that is automatically loaded.

1. Create a `helpers.php` file wherever you like. I usualy keep my global helpers in `app/Support/helpers.php`.
2. Add your your helper functions, no need to specify any namespace *(so we don't have to use `use function` to call them)*.
    
    ```php
    if (!function_exists('to_boolean')) {
    
        /**
         - Convert to boolean
         *
         - @param $booleable
         - @return boolean
         */
        function to_boolean($booleable)
        {
            return filter_var($booleable, FILTER_VALIDATE_BOOLEAN);
        }
    }
    ```

3. In `composer.json` inside the `autoload` section add the following line `"files": ["app/Support/helpers.php"]`.
    
    ```
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Database\\Factories\\": "database/factories/",
            "Database\\Seeders\\": "database/seeders/"
        },
        "files": ["app/Support/helpers.php"]
    }
    ```

4. Run `composer dump-autoload`

Now ower `to_boolean` function is callable anywhere in our project.


Back to ower `Boolean` rule. 

### References

* *[github.com - boolean validation does not accept "true" and "false", but accepts "1", "0"](https://github.com/laravel/ideas/issues/514){:target="_blank"}*
* *[php.net - filter_var - user contributed notes](https://www.php.net/manual/en/function.filter-var.php#121263){:target="_blank"}*
* *[php.net - validate filters](https://www.php.net/manual/en/filter.filters.validate.php#filter.filters.validate){:target="_blank"}*
* *[stackoverflow.com - How do I make global helper functions in laravel](https://stackoverflow.com/a/43243743/4325011){:target="_blank"}*