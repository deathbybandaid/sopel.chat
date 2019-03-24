---
layout: default
title: Sopel 7.0 Migration
---

# Sopel 7.0 Migration

Sopel 7 lays the groundwork for a lot of awesome stuff! We have a major update
to Sopel's command-line interface in progress (which will be finished in Sopel
8), and some API updates that might affect a few existing modules.

But first, we really need to talk briefly about Python.


## A note about Python versions

If you still use Sopel under Python 2, you might notice that Sopel 7 emits
warnings about version compatibility. The sad reality is, as of January 1, 2020,
Python 2.7 no longer receives any updates. While this doesn't mean we will
simply drop support for running Sopel under Python 2 immediately, it _does_ mean
that we will no longer reject ideas that would require doing so.

For the life of Sopel 7.x, we still plan to maintain compatibility with Python
2.7 unless it becomes absolutely necessary to drop it. For example, if a severe
bug is found in one of Sopel's dependencies, and the fix is only released for
Python 3, we would consider dropping Python 2 support in the next minor version.

From Sopel 8 onward, it would be much easier to implement new features and
enhancements if we dropped support for very old Python releases. For example,
Sopel's support for reloading modules during runtime can be made much more
robust using features added to the language in Python 3.4.

The crux of the matter is this: Sopel's range of supported Python releases
remained stagnant for far too long. While the Sopel project was effectively
unmaintained between late 2016 and early 2018, Python 3.3 reached end-of-life
(September 29, 2017). During the lengthy development period of Sopel 7, Python
3.4 reached end-of-life (March 18, 2019). Sopel 7 is likely to be released very
close to the EOL of Python 2.7.

We can't keep testing support for these old versions forever. At some point,
Sopel's core developers will lose the ability to run them locally. (Python 3.3
is already difficult—though not impossible—to install on current popular Linux
systems like Ubuntu 18.04 LTS.) Travis CI, our continuous integration testing
provider for contributions from both maintainers and the community alike, won't
keep supporting the installation of EOL Python releases indefinitely. We can't
support what we can't test.

Keeping all this in mind, the current plan is as follows. Note that it is
subject to change, as Sopel's development pace remains quite leisurely relative
to the overall Python ecosystem.

  - Sopel 7 will try to maintain the same Python version compatibility range as
    Sopel 6
    - This may change as Sopel 7 gets closer to release, depending on how our
      testing infrastructure and dependencies look, but we're motivated to keep
      things as-is (one of Sopel's maintainers still runs a production instance
      on Python 2.7, and cannot upgrade that system to a compatible version of
      Python 3 without significant work)
  - Sopel 8 will drop support for Python releases that are EOL as of the start
    of its development cycle
    - This **definitely** means 2.7, 3.3, and 3.4 (already EOL)
    - Python 3.5 and 3.6 support **might** be dropped, depending on timing
      - Python 3.6 is [tentatively][PEP-494] EOL in December 2021, so presumably
        support for 3.5 will end before then, but we don't have enough concrete
        information from the Python project to _really_ plan this far in advance

  [PEP-494]: https://www.python.org/dev/peps/pep-0494/


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
structure (deprecated in [Sopel 6.2.0][v6.2.0], released January 16, 2016).

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

### Rename/cleanup of `sopel.web`

While the whole `sopel.web` module was marked as deprecated in [Sopel
6.2.0][v6.2.0], because it largely serves as a wrapper around the `requests`
library, parts of it seem to be useful enough that they should be kept around.

For Sopel 8, we intend to move `sopel.web` to `sopel.tools.web`. Ideally the new
location will be available in Sopel 7 to provide a transitional period. Similar
to how importing from both `willie` and `sopel` worked in the run-up to Sopel
6.0, it will be possible to do any of the following during Sopel 7's life cycle:

  - `import sopel.web`
  - `from sopel import web`
  - `import sopel.tools.web`
  - `from sopel.tools import web`

In Sopel 8, we will remove the pointers from `sopel.web` to the new location.
These explicitly deprecated functions will also be removed at the same time:

  - `sopel.web.get()` — use `requests.get()` directly instead
  - `sopel.web.head()` — use `requests.head()` directly instead
  - `sopel.web.post()` — use `requests.post()` directly instead
  - `sopel.web.get_urllib_object()` — seriously, just use [`requests`][requests]

We will also tweak the module constants:

  - `sopel.web.default_headers`: renamed to `sopel.tools.web.DEFAULT_HEADERS`
  - `sopel.web.ca_certs`: removed in `sopel.tools.web` — it no longer has any
    function (and was probably not useful for Sopel plugins to import, anyway)

New additions to Sopel's web tools (there may be a few) will be available only
in the new location (`sopel.tools.web`). Functions and constants that we plan to
remove (as listed above) will be available only from the old `sopel.web` module.

  [requests]: https://pypi.org/project/requests/
  [v6.2.0]: {% link _changelogs/6.2.0.md %}
