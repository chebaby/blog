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

If you have to validate a request input as a boolean, and the input value is `'true'` or `'false'` (notice that I wrapped the values inside single quotes to indicate that those are actually strings) you would expect, *as I did*, that this will just work out of the box and Laravel validator will just see them as booleans. But that is not the case, instead, you will be hit by this beautiful error message `'The inputName field must be true or false.'`

To deeply understand why this is happening, let's go through an example, let's say that we have a form request that we call `PostRequest`.

>Form requests are custom request classes that encapsulate their own validation and authorization logic - Laravel docs.


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

`PostRequest` for simplicity purpose has a single request input that we need to validate as boolean, we will call that `is_published`, and let's say that `is_published` value coming to us from the client is `'true'` or `'false'` ...how can we handle this? 


## Beforehand

Laravel boolean rule doesn't validate `'true'` and `'false'` a valid booleans, you can actually see that in `validateBoolean()` method located in `Illuminate/Validation/Concerns/ValidatesAttributes`.

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

You can also see that in this tests suite.

```php
<?php

public function testValidateBoolean()
{
    $trans = $this->getIlluminateArrayTranslator();
    $v = new Validator($trans, ['foo' => 'no'], ['foo' => 'Boolean']);
    $this->assertFalse($v->passes());

    $v = new Validator($trans, ['foo' => 'yes'], ['foo' => 'Boolean']);
    $this->assertFalse($v->passes());

    // here
    $v = new Validator($trans, ['foo' => 'false'], ['foo' => 'Boolean']);
    $this->assertFalse($v->passes());

    // and here
    $v = new Validator($trans, ['foo' => 'true'], ['foo' => 'Boolean']);
    $this->assertFalse($v->passes());

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
and that's done by using `prepareForValidation()` method. Like its name reveals, this method allows us to add new inputs or update existing ones, which get merged to the request before they go through validation.

So in our small example, we will try to convert the `is_published` value to an actual boolean, and merge it back to the original request.

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
            'is_published' => $this->toBool($this->is_published),
        ]);
    }

    /**
     * Convert to boolean
     *
     * @param $booleable
     * @return boolean
     */
    private function toBool($booleable)
    {
        return filter_var($booleable, FILTER_VALIDATE_BOOLEAN);
    }
}
```

`FILTER_VALIDATE_BOOLEAN` tries to be smart, recognizing words like *`'Yes'`*, *`'No'`*, *`'Off'`*, *`'On'`*, *`'true'`* and *`'false'`*, and is not case-sensitive when validating strings.

`FILTER_VALIDATE_BOOLEAN` returns **true** for *`'1'`*, *`'true'`*, *`'on'`* and *`'yes'`*. Returns **false** otherwise.


### The second approach - use a custom validation rule


### References

* *[boolean validation does not accept "true" and "false", but accepts "1", "0"](https://github.com/laravel/ideas/issues/514){:target="_blank"}*
* *[php.net - filter_var - user contributed notes](https://www.php.net/manual/en/function.filter-var.php#121263){:target="_blank"}*
* *[php.net- validate filters](https://www.php.net/manual/en/filter.filters.validate.php#filter.filters.validate){:target="_blank"}*