# Consuming Modules: Locating Modules in AMD

Ok, so [module ids](004-consuming-modules-module-ids.md) seem simple enough
This sounds straightforward at first, but you're soon left
wondering:

> How does the AMD environment know where to find modules if I'm specifying
ids and not urls?

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
we'll explore an organizational strategy called "packages".

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

>So how do I use modules on a cross-domain server such as a CDN?

<***** put in a snippet to explain that we'll show how to map urls to ids *****>

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

- anonymous modules, when to use them

- rules when loaders revert to url logic
	- curl.js: leading /
	- requirejs: .js extension

- contrast with node id resolution

- pseudo-modules: require, exports, module, global

