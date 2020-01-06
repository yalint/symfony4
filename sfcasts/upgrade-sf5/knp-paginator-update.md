# Upgrading KnpPaginatorBundle & PHP Platform Version

If you look at the list of deprecations, a *bunch* of these all mention the same
thing: some `TreeBuilder::root()` thing. This is a low-level function that
third-party bundles use. And if you dig through the list, this
`stof_doctrine_extensions` comes from StofDoctrineExtensionsBundle... as does
most of the other ones - like `orm`, and `mongodb`. The last one comes from
KnpPaginatorBundle.

So basically... we need to upgrade *both* KnpPaginatorBundle and
StofDoctrineExtensionsBundle.

## Upgrading KnpPaginatorBundle the Lazy Way

Let's start with KnpPaginatorBundle... and let's try to be as *lazy* as possible.
Copy the package name, move over, and run:

```terminal
composer update knplabs/knp-paginator-bundle
```

My *hope* is that a minor upgrade - something like 2.8 to 2.9, which my
`composer.json` version constraints allow - will be enough to fix the deprecation.
And... absolutely *nothing* happens. It didn't upgrade the library at all!

## Digging into a Library's Releases

So much for the lazy way out: *now* we need to do some digging. Google for the
bundle and find their GitHub page. Right now, PhpStorm tells me that we're using
version 2.8.0. Back on the GitHub page, click on "Releases".

Woh! The latest version is 5.0! And it says:

> Added support for Symfony 5

That's what we want! Ok, no problem: to get a version of this library that works
with Symfony 5, we need to upgrade to 5.0. Back in `composer.json`, change the
version to `^5.0`.

And yes, because we're upgrading to a new *major* version - heck, we're upgrading
*3* new major versions - this *could* contain some backwards-incompatible changes
that will break our app. Let's... worry about that in a little while.

Let's update this!

```terminal
composer update knplabs/knp-paginator-bundle
```

## Checking your config.platform.php Setting

And... this fails. Boo! Let's see: we tried to get version 5.0 of the bundle... but
it requires PHP 7.2 or higher... and my PHP version is 7.3.6... but is overwritten
by my config.platform.php version, which is 7.1.3.

Wow! That's fancy way of saying that version 5 of this library requires a higher
version of PHP than I'm using. Except... well... that's not completely true.
I'm using PHP 7.3, but in my `composer.json` file... if you search for `config`,
here it is: I added a `config.platform.php` key set to 7.1.3. This is optional and
you might not have this in your app... but I *do* recommend having it. It tells
Composer to *pretend* like I'm using PHP 7.1.3 when I'm downloading dependencies
*even* though I'm *actually* using PHP 7.3.

By setting this value to whatever PHP version you have on your production server,
it'll make sure you don't accidentally download any packages that work on your
local machine by explode on production.

So if our goal is to upgrade to Symfony 5, our production server will need to
*at least* be set to Symfony 5's minimum PHP version, which is 7.2.5. And
for consistency... even though it doesn't affect anything, under the `require`
key, update this too.

## Updating --with-dependencies

Ok *now* that Composer knows it's ok to download packages that require PHP 7.2...
let's try that update command again:

```terminal-silent
composer update knplabs/knp-paginator-bundle
```

And... yay! Another error! I mean, another *fascinating* challenge that we are
*totally* up to crushing. Let's see: KnpPaginatorBundle requires something called
`knp-components` and... basically 5.0 of the bundle requires version 2 of the
`knp-components`, but we're currently "locked" at version 1.3. That means that
we currently have 1.3 installed.

This `knp-components` library is *not* something that we have directly in our
`composer.json` file: it's a "transitive" dependency, which is a hipster way
of saying that our app only needs it because KnpPaginatorBundle needs it.

So then... why didn't Composer just update *both* libraries? Because Composer is
conservative: we told it to *only* upgrade `knplabs/knp-paginator-bundle` and
correctly figured out that it can't *only* upgrade that *one* package.

To fix this, run the command again but now add `--with-dependencies`:

```terminal-silent
composer update knplabs/knp-paginator-bundle --with-dependencies
```

This says: it's ok to upgrade `knp-paginator-bundle` *and* also any of *its*
dependencies. This time... it did the trick: this upgrades from version 1 to 2
of `knplabs/knp-components` and from version 2 t 5 of `knplabs/knp-paginator-bundle`.

## Checking Major Upgrade Changelogs

Awesome! Except... we need to be careful: these are *major* version upgrades...
which means that they might contain "breaking" changes.

Go back to the GitHub homepage for KnpPaginatorBundle and look for a `CHANGELOG.md`
file. Not *every* library will have this... but hopefully most do. Let's see:
the breaking changes for version 3 were just removing support for old PHP versions.
For version 4... it dropped support for old PHP and old Symfony versions... and
for version 5, it added a return type to `PaginatorAwareInterface`... which is
not something I'm using in my app - you could search the code to be sure.

So... we're good! You could repeat this for the `knp-components` library if you
want, though since we're not using its code *directly* in our app, we should be
good.

Ok, we've handled this `knp_paginator` deprecation. Next, let's update
StofDoctrineExtensions bundle to remove the *rest* of the "tree" deprecations.