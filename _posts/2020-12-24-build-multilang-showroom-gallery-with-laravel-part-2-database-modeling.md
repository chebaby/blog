---
layout: post
title: Building multilang show room gallery with laravel
subtitle: Database Modeling
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel]
comments: true
---

Before getting our hands dirty with database, first let's list all possible table that we gon need to build our project.

Busines rules

Website is multilang, with enlish and french as languages of choise.  
Product could fall into one or many categories.  
Categories could have many sub categories (deep to one level) wich also mean that.  
Category could from none to one parent.  

* Artisans
* Categories
* Products

since this is a multilang website, we gon to need to other table where to stock localized fields. with that in mind no we have this:

* Artisans
* Artisan Translations
* Categories
* Category Translations
* Products
* Product Translations


let's jump in to modeling our database, 


