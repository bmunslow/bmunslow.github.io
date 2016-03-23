---
layout: post
comments: true
title: Why I avoid Drupal's maintenance mode and you probably should too
---

Drupal 7 ships with a quick and easy to setup maintenance mode but I don't recommend using it at all.  Keep reading to find out why.

-----

You would expect Drupal's maintenance mode to be a safe haven which allows you to perform common maintenance tasks such as:

* Reverting a few features
* Updating modules or Drupal core
* Install and enable new modules
* Etc.

These tasks are often intensive on the database and CPU usage of your server, so you don't want any users to be browsing your site and potentially halt this tasks.

It turns out to be Drupal's maintenance mode is not safe at all!  (Yes, I had to learn this the hard way...).

The reason why is that when a visitor loads an uncached page in your site, EVEN if your Drupal site is on maintenance mode, `hook_init` is triggered, causing other modules to possibly try to update the `cache_field` table (or other ones).

If you happen to be running any of the aforementioned tasks while this happens, you are very likely to get a number of dreaded MySQL Deadlocks, which will likely interrupt the whole process.

And you don't want that!

It can easily lead to corrupt database states and other nasty scenarios (such as Mysql `error 1051`, which might require you to [drop and rebuild all your databases!](http://stackoverflow.com/questions/3927690/howto-clean-a-mysql-innodb-storage-engine/4056261#4056261)).

This issue has [already been reported on Drupal.org](https://www.drupal.org/node/2523880) but it is lying in the depth's of the Drupal 7 core issues pool and it doesn't look like it's getting much attention at the moment.

## The alternative

There are a number of solutions to this issue.  I personally use the following snippet to force the server to completely avoid hitting Drupal at all, and just accept requests to a static `HTML` file, indicating the site is not available at the moment.

Just drop the following lines at the end of your `.htaccess` file (provided your server runs on Apache) and create an .html in the root of your Drupal installation.

{% gist a032080627e0230cfd8c %}
