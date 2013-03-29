# Consuming AMD Modules

As we discussed briefly in
[Authoring AMD Modules](001-authoring-amd-modules.md), some modules require
other modules to do their work.  The module author specifies the other
modules by listing each module's *id* in the dependency list or in a
"local require".  This sounds straightforward at first, but you're soon left
wondering:

> How does the AMD environment know where to find modules if I'm specifying
ids and not urls?

We'll answer that question, but first let's investigate module ids a bit more.

## Module Ids

AMD and CommonJS both specify module ids that look very much like file paths or
urls: ids consist of *terms* separated by slashes.  The definition of "terms"
is fairly loose.  The CommonJS
[spec](http://wiki.commonjs.org/wiki/Modules/1.1#Module_Identifiers) further
restricts "terms" to be camelCase Javascript identifiers, but in practice,
other popular file name characters, such as `-` are perfectly acceptable.  The
proposed ES6 modules
[spec](http://wiki.ecmascript.org/doku.php?id=harmony:modules) is much more
flexible, but realistically the terms should be strings of chars that can be
used in file systems and urls.

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
have ids of the form "curl/<submodule>", but don't exist on the file system.

Be careful to capitalize correctly.  Since most modules are mapped to urls
-- and urls often map to files -- the module name should be spelled and
capitalized exactly the same as the file name.  For instance, "jQuery" is
almost always *not* the correct module id.

## Relative Ids

AMD and CommonJS also support the notion of *relative* module identifiers.
Modules that reside in the same hierarchical level can be referenced by using
 `./` at the beginning of the id.  Modules that reside one level up
from the current level can be referenced using `../`.

At run time or build time, the AMD environment must translate relative ids
to *absolute* ids.  Absolute ids are "rooted" at the top level of the module
hierarchy and contain no `..` or `.`.  The process of removing the leading
`..` or `.` is called "normalization".  For example, using
"app/billing/billTo/Customer" as the id of our current module, the environment
will normalize the following ids:

* "./store" normalizes to "app/billing/billTo/store"
* "../payee/Payee" normalizes to "app/billing/payee/Payee"

AMD and CommonJS also recognize bare `.` and `..` as module identifiers.  `.`
normalizes to the module whose name is the same as the current level. `..`
normalizes to the module whose name is the same as the level that is one
level up from the current level.  Yes, that is confusing!  Perhaps that's
why you don't see these used often.  Hopefully, some examples might help.
Given that the current module is "app/billing/billTo/Customer", the following
are true:

* "." normalizes to "app/billing/billTo" (a module, not a folder!)
* ".." normalizes to "app/billing" (a module, not a folder!)

_Hint:_ never use relative module ids to reference unrelated modules!  More
than one set of `../` may be a code smell that you need to better organize
your modules.  The relative id may also be interpreted as a url, rather than
an id by the AMD environment.  See the section "Why multiple `../` is a
code smell" for more information.


## Default Module Location and Base Url

AMD environments must locate modules.  In browsers, this means that the AMD
environment must somehow resolve the id into a url.  For extremely simple
applications, you could put all of the modules in one location. Call this
location the "base url".  The method for resolving module ids could then
just be simple string concatenation:

```
module url = base url + module id + ".js"
```

By default, most AMD loaders set the base url to the location of the html
document.  For instance, if the html document is at
http://know.cujojs.com/index.html, then a module whose id is "blog/controller"
would reside at http://know.cujojs.com/blog/controller.js.  The ".js"
extension is automatically added when the AMD environment resolves the url.

Keeping your modules in the same location as your html documents could be
inconvenient.  Therefore, virtually all AMD environments allow the base url
to be set via configuration.  For instance, if we configured our base url to
be "client/apps", then the module id, "blog/controller", would resolve to
http://know.cujojs.com/client/apps/blog/controller.js.

### Module ids != urls

It's very easy to get started by setting the base url and putting a few
modules in that folder, but don't get lured into thinking that module ids
are simply shortened urls!  This pattern fails to scale beyond trivial
examples.  Real apps require organizational strategies.  In the next tutorial,
we'll explore an organizational strategy through an abstraction
called "packages".

### Some times id == url, no?

> What if my module requires a library on a CDN?

Most AMD loaders allow urls to be specified in place of ids.  This is perfectly
valid:

```js
define(function (require) {
	var moment = require('//cdnjs.cloudflare.com/ajax/libs/moment.js/2.0.0/moment.min.js');
	// do something interesting
});
```

Well, there are several problems with this code:

1.	Hard-coding urls in code limits maintainability. What if you want to update
	to the latest version?  (AMD provides the means to configure id-to-url
	mappings.  This is covered in the next tutorial.)
2.	The ".js" extension can trigger some AMD environments to use legacy,
	non-module behavior.  RequireJS, for instance will do this.
3.	Some AMD-aware libraries have hard-coded ids into their files,
	unfortunately.  Moment.js, for instance, hard-coded the id, "moment"
	into their file.  They're essentially squatting on this name.  Even worse,
	this means that the AMD environment fetched a module named
	"//cdnjs.cloudflare.com/ajax/libs/moment.js/2.0.0/moment.min.js", but
	 received a module named "moment".  The AMD environment will likely
	 throw an error.

Are you sure you still want to use urls? :)

## Why multiple `../` is a code smell

To the AMD environment, the base url determines the "root" or "top level" of
the location of the module hierarchy.  Attempting to *traverse* above the
root doesn't make much sense.  For instance, consider the following scenario:

* Module "blog/controller" resolves to
  "http://know.cujojs.com/client/apps/blog/controller.js".
* Module "blog/controller" requires a module with id of "dont/please", but
  refers to it *relatively* as "../../dont/please".

In order to resolve this relative id, the AMD environment would have to
traverse up two levels before traversing back down to "dont" and "please".
Traversing up one level from "blog/controller" (we're at the "blog" level
at this point) yields "" (the root).  How do we traverse above that?

How AMD environments handle this situation is *not defined*.  curl.js resolves
it by assuming the id is actually a url.  The implication of this is that
curl.js will load the module, but will likely not recognize that "dont/please"
and "../../dont/please" are the same module.  This could cause double-loading
of the "dont/please" module. Furthermore, the module's factory could execute
twice, causing all sorts of problems, including singletons to no longer be
single, etc.

If you find yourself needing to reference other modules using multiple `../`,
be sure to read the next tutorial about packages!

-----

Need to cover these topics, too:

- anonymous modules

- rules when loaders revert to url logic
	- curl.js: leading /
	- requirejs: .js extension

- contrast with node

- pseudo-modules: require, exports, module

