---
layout: post
title: Building multilingual showroom gallery with laravel - part 2
subtitle: Database Modeling
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, database]
comments: true
---

Before getting our hands dirty with database migrations, first let's list all possible table that we gon need to build our project, for that, here's some some basic business rules to guide us:

Website is multilingual, with enlish and french as languages of choise.  
Product belongs to one or many categories.  
Categories could have many sub categories (deep to one level).  
Category could have no parent or one parent at max.

The given rules above, could easly accomplished with these tables:

* Artisans
* Categories
* Products

But, since this is a multilingual website, we gon to need other tables to store localized fields. For that I plan to use 
[mcamara/laravel-localization](https://github.com/mcamara/laravel-localization) package. This package, as the you can read on the ReadMe, offers the following:

* Detect language from browser
* Smart redirects (Save locale in session/cookie)
* Smart routing (Define your routes only once, no matter how many languages you use)
* Translatable Routes
* Supports caching & testing
* Option to hide default locale in url
* Many snippets and helpers (like language selector)

With that in mind, we end up with this:

* Artisans
* Artisan Translations
* Categories
* Category Translations
* Products
* Product Translations
* Product Categories

I think this is enough to get the application up and running, other tables may be added along the way as needed, but let's keep it simple for the time been.

Notice that there is new table named "Prodcut Categories", that's the result of the many-to-many relationship between "Product" table and "Categories" table, more on that later.

## Migrations
