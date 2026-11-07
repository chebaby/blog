---
layout: post
title: Convert Symfony/twig localization to Laravel/blade localization
share-description: I will talk about my experience while migrating localized twig views from Symfony 2.6 (php5.4) to localized blade views in Laravel 8 (php7.4).
gh-repo: chebaby/blog
gh-badge: [follow]
tags: [laravel, localization, blade, symfony, twig]
comments: false
published: false
head-extra: article_structured_data.html
---

We had this relativly old website that is build with Symfony framework version 2.6 on top of an old php version, version 5.4, we wanted to upgrade this website from Symfony to Laravel, so re-build 90% from scratch, the other 10% is just some views, that were nicly done, and are not heavly dynamic, mostly static pages, so we dicided we are just gone to copy/past those views from Twig to Blade views. 

As I montioned before those views are mostly static pages and there is only a fiew of them, So, one valid solution is to also import those translations from messages.xliff file to locale.json file and in the views replace twig translation to it's equivalent in blade, for example {% raw %}`{{ 'hello, world'|trans }}`{% endraw %} to {% raw %}`{{ __('hello, world') }}`{% endraw %}, and we have ower self a working localized website.

But, after a fiew minutes of manualy copying and pasting localiazation keys, and replacing Twig `trans` with Blade `trans`,  my inner "developer-leaziness" kicked in, it told me this is not how a "real" developer would solve this, a "real" developer would not do the same simple task over and over again like a robot, when his has all the power to just build it's own robot to do job in no time and with efficiency.

So, long stroy short I well share with you my robot (php script)




### References

* *[php.net](https://www.php.net){:target="_blank"}*