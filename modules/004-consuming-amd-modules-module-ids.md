# Consuming AMD Modules: Module Ids

As we discussed briefly in
[Authoring AMD Modules](001-authoring-amd-modules.md), some modules require
other modules to do their work.  The module author specifies these other
modules by listing each module's *id* in the dependency list or in a
"local require".  This sounds straightforward at first, but you're soon left
wondering:

> How does the AMD environment know where to find modules if I'm specifying
ids and not urls?

We'll answer that question [soon](), but first let's investigate module ids a bit more.

## Module Ids

AMD and CommonJS both specify module ids that look very much like file paths or
urls: ids consist of *terms* separated by slashes.  The definition of "terms"
is fairly loose.  The CommonJS
[spec](http://wiki.commonjs.org/wiki/Modules/1.1#Module_Identifiers) further
restricts "terms" to be camelCase Javascript identifiers, but in practice,
other popular file name characters, such as `-` are perfectly acceptable.  The
proposed ES6 modules
[spec](http://wiki.ecmascript.org/doku.php?id=harmony:modules) is much more
flexible, but, realistically, ids should be compatible with file systems
and urls.

AMD reserves the `!` character to indicate that a
[Loader Plugin](https://github.com/amdjs/amdjs-api/wiki/Loader-Plugins) should
be used to load the module (or other type of resource).

Some examples of acceptable module ids:

```
"wire/lib/functional"
"poly/es5-strict"
"app/billing/billTo/Customer"
"jquery"
```

As with file systems and urls, the slashes delineate organizational
hierarchies.  Typically, these hierarchies are mirrored by identical
directory structures in the underlying file system, but this isn't guaranteed.
For example, curl.js exposes its extensibility API as modules.  These modules
have ids of the form "curl/<submodule>", but they don't actually exist as
files.

Be careful to capitalize correctly.  Since most modules typically map
to files, the module name should be spelled and capitalized exactly the
same as the file name.  For instance, "jQuery" is almost always *not*
the correct module id (capital "Q")!

## Relative Ids

AMD and CommonJS also support the notion of *relative* module identifiers.
Modules that reside in the same hierarchical level can be referenced by using
 `./` at the beginning of the id.  Modules that reside one level up
from the current level can be referenced using `../`.

At run time or build time, the AMD environment must translate relative ids
to *absolute* ids.  Absolute ids are "rooted" at the top level of the module
hierarchy and contain no `..` or `.`.  The process of removing the leading
`..` or `.` is called "normalization".  For example, assuming
"app/billing/billTo/Customer" is the id of our current module, the environment
will normalize required ids as follows:

```js
define(function (require) {
	// normalizes to "app/billing/billTo/store"
	var store = require("./store");
	// normalizes to "app/billing/payee/Payee"
	var Payee = require("../payee/Payee");
});
```

AMD and CommonJS also recognize bare `.` and `..` as module identifiers.  `.`
normalizes to the module whose name is the same as the current level. `..`
normalizes to the module whose name is the same as the level that is one
level up from the current level.  Yes, that is confusing!  Perhaps that's
why you don't see these used often.  Hopefully, some examples might help.
Given that the current module is "app/billing/billTo/Customer", the
environment will normalize these ids as follows:

```js
define(function (require) {
	// normalizes to "app/billing/billTo" (a module, not a folder!)
	var billTo = require(".");
	// normalizes to "app/billing" (a module, not a folder!)
	var billing = require("..");
});
```

_Hint:_ never use relative module ids to reference unrelated modules!  Relative
modules are meant to be used *within* a "package" (defined later).  Also,
more than one set of `../` may be a code smell that you need to better organize
your modules.  The relative id may also be interpreted as a url, rather than
an id by the AMD environment.  See the section [Why multiple `../` is a
code smell](#why-multiple--is-a-code-smell) for more information.
