---
layout: default
title: Sopel 7.0 Migration
---

# Sopel 7.0 Migration

Sopel 7 lays the groundwork for a lot of awesome stuff! We have a major update
to Sopel's command-line interface in progress (which will be finished in Sopel
8), and some API updates that might affect a few existing modules.


## CLI restructuring

Version 7 deprecates most of the command-line arguments that Sopel has used for
most of its life, in favor of a much more extensible command structure.

Instead of having arguments like `--quit` or `--configure-modules`, Sopel's CLI
now works on a sub-command system (like Git and other popular tools). This will
allow adding more functions without having to share one global argument
namespace, and make the commands more "speakable".

The full new command structure will be documented in the [Command-line
arguments]({% link _usage/command-line-arguments.md %}) after release, but as a
general overview of the old structure vs. the new:

|             This            |           Becomes           |
| :-------------------------- | :-------------------------- |
| `sopel --quit`              | `sopel quit`                |
| `sopel --kill`              | `sopel quit --kill`         |
| `sopel --configure-all`     | `sopel configure`           |
| `sopel --configure-modules` | `sopel configure --modules` |

There's one argument deserving of special mention: `--migrate`/`-m`. It will
be removed in Sopel 7, because it has been a no-op since version 4.0.0.
Someone deleted the code to handle it without mentioning it anywhere, but
Sopel has continued to carry around this useless argument since 2014. No more!

New commands are not always going to be shorter than the old ones (see the
`--kill` example), but we're looking at the future picture. This is just the
start of Sopel's CLI evolution, and more features will come based on this new
command structure.

### Removal of old commands

For the life of Sopel 7, the existing ("legacy") arguments are still supported,
just with deprecation notices in the `--help` output.

In Sopel 8, the old arguments from Sopel 6 and lower will be removed.

Later releases of Sopel 7 may output warnings when deprecated arguments are
used, but you really should update your scripts immediately upon upgrading to
Sopel 7. Then you can't forget later!


## Sopel 7 API changes

### Managing URL callbacks

For quite a while, Sopel modules wishing to override the `url.py` module's
automatic title-fetching for certain URLs have customarily done something along
these lines:

```python
# in the module's setup() function:
    if not bot.memory['url_callbacks']:
        bot.memory['url_callbacks'] = tools.SopelMemory()
    bot.memory['url_callbacks'][compiled_regex] = methodname
```

Similar manual manipulation of the object in memory was needed to unregister
handlers at module unload:

```python
# in the module's shutdown() function:
    try:
        del bot.memory['url_callbacks'][compiled_regex]
    except KeyError:
        pass
```

Going forward, a new set of API methods should be used instead:

  - `bot.register_url_callback(pattern, methodname)`, to call `methodname` when
    a URL in a message matches the `pattern`
  - `bot.unregister_url_callback(pattern)`, to unregister the `pattern` and its
    associated callback(s)
  - `bot.search_url_callbacks(url)`, to find callbacks matching the given `url`

Manually accessing `bot.memory['url_callbacks']` as before will continue to work
for the life of Sopel 7.x, at a minimum. However, doing so is considered
deprecated, leaving future versions free to move the callback storage if needed.


## Planned future API changes

### Removal of `bot.privileges`

Sopel 7.x will be the last release series to support the `bot.privileges` data
structure (deprecated in [Sopel 6.2.0]({% link _changelogs/6.2.0.md %}),
released January 16, 2016).

Beginning in Sopel 8, `bot.privileges` will be removed and modules trying to
access it will throw an exception. `bot.channels` will be the _only_ place to
get privilege data going forward.

We suggest updating your own modules as soon as possible, if you have not
already done so, to avoid forgetting later (this will be a theme for these
notes). Updating modules published to PyPI should take priority, especially
modules written for Sopel 6 that are not future-proofed by capping Sopel's
version in their requirements.

If you use third-party modules that have not been updated, we encourage you to
inform the author(s) politely that they need to update. Or better yet, submit
a pull request or patch yourself!
