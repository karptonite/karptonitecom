---
title: "Introducing GeekCache"
date: 2014-07-28T21:55:07-04:00
categories:
    - php
    - caching
---

I'm happy to announce the beta release of my first open source project,
[GeekCache](https://github.com/karptonite/geekcache). GeekCache is a PHP
library for key/value storage (currently implemented for Memcached). Why
release a new caching library, when excellent libraries such as
[Stash](http://www.stashphp.com/) and
[doctrine/cache](https://packagist.org/packages/doctrine/cache) are already out
there? Because I needed features that those libraries did not offer. GeekCache
is a complete rewrite of a caching library we've used internally at
[BoardGameGeek](http://boardgamegeek.com) for years. We are dependent on its
features, some of which are entirely unavailable in other libraries (as far as
I know), and certainly not all in the same library.

In addition to the basic features you are likely to find in almost any caching
library, here are the key features that distinguish GeekCache:

* Invalidation via tags
* Memoization
* Regeneration via pass-through regenerator callables
* Soft invalidation, for returning stale data when regenerating through queued processes.
* a fluent interface for cache item creation

Below are a few quick examples. For more details, including how to use the
included service provider to get a the cache builder and clearer, see the
[GeekCache GitHub page](https://github.com/karptonite/geekcache).

Basics
------

I most often use GeekCache to cache the results of MySQL queries.  On a busy
site such as BoardGameGeek, it would be very slow to pull an item from the
database every time we needed it. Instead, we almost always store the results
of a query in cache, so that the query itself is run only once.

But storing in cache is the easy part. The hard part is making sure that the
cache is cleared when it is no longer valid. This is where tags come in.

Tags
----

We can add any assortment of tags when storing a value to cache. When the tag is
cleared, all items with that tag are cleared as well. Say we are retrieving
user 1 from the database.

~~~php
$cacheitem = $cacheBuilder
    ->memoize()
    ->addTags('users', "user_1")
    ->make("tag_user_1", 3600);

$user = $cacheitem->get();
// $user is false if it has not been stored in cache

if ($user === false) {
    $user = $this->getUser(1);
    $cacheitem->put($user);
}

//later

$result = $cacheitem->get();
// result equal to $user

// now, the user object is changed. The cache must be cleared.
$cacheClearer->clearTags("tag_user_1");

$result = $cacheitem->get();
// result is now false, because the cache has been cleared
~~~

Memoization
-----------

When you add `memoize()` to a chain of build methods, cache values will be
stored in a local array cache for the duration of a php process. If a cache
item is retrieved a second time over the course of the page load, the value
from the local cache will be returned. This can substantially improve
performance if some values are looked up multiple times on a page load.


Regenerators
------------

Regenerators are quite powerful. Not only do they make caching code cleaner, by
allowing the process that regenerates the cache to be handled by GeekCache (via
a closure or any other callable), they can also allow you to return stale data
to users (rather than blanks) when the process to regenerate the data is too
slow to be run on page generation.

~~~php

$cacheitem = $cacheBuilder
    ->memoize()
    ->addTags('users', "user_1")
    ->make("tag_user_1", 3600);

$regenerator = function () use ($this, $userid) {
    return $this->getUser($userid);
}

$value = $cacheitem->get($regenerator);
// $value is the result of getUser

$value = $cacheitem->get();
// $value is the correct result, because the regenerated value has been put into cache

$cacheClearer->clearTags("user_1");

$value = $cacheitem->get();
// $value is now false, because one of the tags has been cleared 

$queuedRegenerator = function() use ($this, $userid) {
    // queue up a process to regenerate the results in another process
    // returning false indicates that a process has been queued, so any stale data
    // that might be available is returned
    return false;
}

$value = $cacheitem->get($queuedRegenerator);
// $value is the original cached user
// Tag are cleared via soft invalidation, so that stale data can be
// returned, if and only if a process is spun off to regenerate the cache 
~~~

See the documentation for how to add a grace period, so that stale data can
also be returned from caches that have expired due to time.

Other Stuff
-----------

This is just some of what GeekCache has to offer. In addition to other
features, such as a counter (including atomic incrementing), GeekCache also
offers fully tested, extensible code. Except where absolutely necessary, all
classes are immutable--no need to worry about whether reusing a builder will
mean bleed over from a previous use. 

I'd love to get some constructive feedback on places people think
GeekCache could be further improved, especially (but by no means exclusively)
from people who might want to use it in their own projects. In particular, I'm
still not fully satisfied with some of the naming choices I've made. Take a
look under the hood, and tell me what you think.

I made the GeekCache code and the interfaces as clean and expressive as I was
able, and while I still see room for improvement in places, on the whole, I'm
pleased with the way it's turning out. I would be quite happy to hear if anyone
else found a use for it in their own projects!
