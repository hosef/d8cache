# Drupal 8 Cache Backport (D8Cache)

This guide contains information on the user-configurable parts of D8Cache.
Fully utilizing D8Cache may require some site customization. For clarity,
developer-centric documentation has been split out to `README.developer.md`.

Introduction - How does tag based caching work?
-----------------------------------------------

With Cache Tags it is possible to tag pages or other cache entries like 'blocks'
by using so-called cache tag strings. (e.g. `node:1`).

Then when the `node:1` object is updated the tag is invalidated.

The D8Cache module already supports cache tags for entities, menus, blocks and
search out of the box.

Getting started
---------------

To get started you download the module and enable it.

Then you need to configure settings.php:

```
$conf['cache_backends'][] = 'sites/all/modules/d8cache/d8cache-ac.cache.inc';
$conf['cache_class_cache_views_data'] = 'D8CacheAttachmentsCollector';
$conf['cache_class_cache_block'] = 'D8CacheAttachmentsCollector';
$conf['cache_class_cache_page'] = 'D8Cache';
```

In this example both `cache_views_data` and `cache_block` are taken over by a
`D8CacheAttachmentsCollector` class, which will ensure that render array
`#attached` properties are tracked properly when performing render caching.

Performance and Reliability notes
---------------------------------

- Possible deadlock when saving content

If you are storing cache tags in the database, be aware there is a
[D8 core issue](https://www.drupal.org/project/drupal/issues/2966607)
(#2966607) open regarding a possible deadlock between content saving and tag
invalidation. If you are encountering deadlocks when saving content, you may
want to try using the Drupal 7 backport of this patch from
[here](https://www.drupal.org/project/drupal/issues/3004437) -- it adds the
ability for modules to run code after a database transaction has been committed.

A future Drupal 7 release will include this functionality, once the Drupal 8
version has been finalized.

D8Cache will automatically detect when this support is available, and if so,
will defer tag invalidation until after the current transaction has completed.

- Drupal core patch may be needed for accurate render caching

If you are having trouble with missing javascript / CSS when doing render
caching, try running this the core patch from
[here](https://www.drupal.org/project/drupal/issues/2820757) (#2820757.)

See that issue for specifics on when this may be necessary.

Frequently asked questions - Users
----------------------------------
For answers to more code-related questions, please see the corresponding FAQ in `README.developer.md`.

### How can I specify a class to use as backend for D8Cache if my bin uses a different backend from the default?

Before:

```
$conf['cache_default_class'] = 'Redis_Cache';
$conf['cache_class_cache_page'] = 'DrupalDatabaseCache';
```

After:

```
$conf['cache_default_class'] = 'Redis_Cache';
$conf['cache_class_cache_page'] = 'D8Cache';
$conf['d8cache_cache_class_cache_page'] = 'DrupalDatabaseCache';
```

So the only thing needed is to prefix the usual 'cache_class_*' variable with
'd8cache_'.

It is also possible to specify another default backend for D8Cache with
'd8cache_cache_default_class', which can be useful to add another decorator just
to the cache bins using the D8Cache backends.

Note that D8Cache is generally only useful for cache bins that store rendered
content. While it can be used on other cache bins with no issues, it is
slightly more efficient to use a regular cache backend class for most bins.

### How I store cache tags in a different backend than the database?

Add this to your settings.php:
```
$conf['d8cache_use_cache_tags_cache'] = TRUE;
```

This will make D8Cache look in the 'cache_d8cache_cache_tags' bin for the cache
tags, and only check the database for any tags that it could not find in this
secondary cache.

Note for developers:
Alternatively, you can extend the `D8Cache` and `D8CacheAttachmentsCollector`
classes and use a trait to override the `checksumValid()` and `getCurrentChecksum()`
functions. Your module should also implement `hook_cache_tags_invalidate()`.

### How do I serve the page cache without boostrapping the database?

Add to settings.php:
```
$conf['d8cache_use_cache_tags_cache'] = TRUE;
// D8Cache will automatically bootstrap the database if needed.
$conf['page_cache_without_database'] = TRUE;
// Unless your boot modules were specifically designed to work without the
// database, you must disable page cache hooks. (hook_boot/hook_exit)
$conf['page_cache_invoke_hooks'] = FALSE;
// Use D8Cache for cache_page to handle tag invalidation properly.
$conf['cache_class_cache_page'] = 'D8Cache';
// cache_bootstrap is needed during Drupal startup, and must use a cache
// backend directly.
$conf['cache_class_cache_bootstrap'] = 'MemCacheDrupal';
// cache_d8cache_cache_tags must also use a cache backend directly.
$conf['cache_class_cache_d8cache_cache_tags'] = 'MemCacheDrupal';
// This is the backing store for cache_page. Invalidation will be handled
// by D8Cache as configured above.
$conf['d8cache_cache_class_cache_page'] = 'MemCacheDrupal';
```
The database will still be bootstrapped if any of the cache tags are missing
from `cache_d8cache_cache_tags`. This is necessary to ensure that invalidations
will be honored properly even when some of the cache tags have been expired from
the cache tags cache.

### Every content change expires all pages. How can I avoid that?
### How can I emit a custom header with cache tags for my special Varnish configuration?
### How can I add a custom cache tag?
### How can I avoid the block and page cache being invalidated when content changes?
### When does max-age "bubble up" to containing cache items and when does it not?

See the corresponding question in README.developer.md.

### Is there more material to learn about cache tags and cache max-age?

The official documentation for Drupal 8 is the best resource right now:

* https://www.drupal.org/docs/8/api/cache-api/cache-tags
* https://www.drupal.org/docs/8/api/cache-api/cache-max-age

The backported API for Drupal 7 should be as similar as possible. See also
the section "Drupal 8 API comparison" in `README.developer.md`.

### My question is not answered here, what should I do?

Please open an issue here:

  https://www.drupal.org/project/issues/d8cache
