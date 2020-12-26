---
layout: post
title: Building multilang show room gallery with laravel - part 2
subtitle: Database Modeling
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, database]
comments: true
---

Before getting our hands dirty with database migrations, first let's list all possible table that we gon need to build our project, for that, here's some some basic business rules to guide us:

Website is multilang, with enlish and french as languages of choise.  
Product belongs to one or many categories.  
Categories could have many sub categories (deep to one level).  
Category could have no parent or one parent at max.

The given rules above, could easly accomplished with these tables:

* Artisans
* Categories
* Products

But, since this is a multilang website, we gon to need other tables to store localized fields. With that in mind, we end up with this:

* Artisans
* Artisan Translations
* Categories
* Category Translations
* Products
* Product Translations

I think this is enough to get us up and running, other tables may be added along the way, but let's keep it simple for the time been. we gon add more as we need it




