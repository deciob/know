# Consuming Modules: Locating Modules in AMD

Ok, so [module ids](004-consuming-modules-module-ids.md) seem simple enough:
slashes delineate terms, which represent hierarchies of modules.  Ultimately,
though, AMD environments must locate modules.  The ids must somehow resolve
to uris.  It's possible that the uris resolve to records in a database
or values in localStorage, for instance.  However, most of the time, the
uri will resolve to a file path (on the server) or a url (in the browser).

## Default Module Location and Base Url

For extremely simple applications, you could put all of the modules in
one location. Call this location the "base url".  The method for resolving
module ids could then just be simple string concatenation:

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
modules in that folder, but don't be lured into thinking that module ids
are simply shortened urls!  This pattern fails to scale beyond smallish
apps.  Larger apps require organizational strategies.  In another tutorial,
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

However, there are several problems with this code:

1.	Hard-coding urls in code limits maintainability. What if you want to update
	to the latest version?
2.	The ".js" extension can trigger some AMD environments to use legacy,
	non-module behavior.  RequireJS, for instance will do this.
3.	Some AMD-aware libraries have hard coded ids into their files,
	unfortunately.  Moment.js, for instance, hard-coded the id, "moment"
	into their file.  They're essentially squatting on this name.  Even worse,
	this means that in the example above, the AMD environment fetched a
	module named
	"//cdnjs.cloudflare.com/ajax/libs/moment.js/2.0.0/moment.min.js", but
	received a module named "moment".  The AMD environment will likely
	throw an error because the ids didn't match.

> So how do I use modules on a cross-domain server such as a CDN?

<***** put in a snippet to explain that we'll show how to map urls to ids *****>

## Why multiple `../` is a code smell

To the AMD environment, the base url determines the "root" or "top level" of
the location of the module hierarchy.  Attempting to *traverse* above the
root doesn't make much sense.  For instance, consider the following scenario:

```js
// module blog/controller @ //know.cujojs.com/client/apps/blog/controller.js
// since base url is //know.cujojs.com/client/apps/
define(function (require) {
	var dont = require("../../dont/please");
});
```

In order to resolve this relative id ("../../dont/please"), the AMD
environment would have to traverse up two levels before traversing back
down to "dont" and "please".

1. Start at the current level: "blog/controller" is at the "blog/" level
2. Traverse up one level: "blog/" --> "" (the top level)
3. Traverse up a second level: ???? (we can't, we're already at the top!)

How AMD environments handle this situation is *not defined* in any spec.
curl.js resolves it by assuming the id is actually a url and keeps the `../` so
the url can be  normalized against the base url.  The implication is that
curl.js will load the module as usual, but will not recognize that "dont/please"
and "../../dont/please" are the same module.  This could cause double-loading
of the "dont/please" module. Furthermore, the module's factory could execute
twice, causing all sorts of problems, including singletons to no longer be
single, etc.

_Note:_ curl.js also assumes that ids that start with `/` or a protocol
(`http:`, `https:`, etc.) are urls.

If you find yourself needing to reference other modules using multiple `../`,
be sure to read the tutorial about packages!  Packages solve everything.
Seriously.  They do.

## Configuring id-to-uri mappings

Ultimately, you have to tell the AMD environment how to map ids to uris.
This is called _path mapping_ or _package mapping_ and is done through
configuration.

_By specifying the urls in a central configuration, instead of inside your
modules, you are decreasing maintenance costs and increasing the portability
of your code._

Here's what the configuration looks like in most AMD environments:

```js
var config = {
	baseUrl: "client/apps",
	paths: {
		"blog": "blog", // this is redundant, unnecessary
		"dont": "../../dont"
	}
};
```

In curl.js, you set the configuration using the global `curl` variable.
Lots of other AMD environments use this API, too:

```js
// auto-sniff for an object literal:
curl(config);

// or, more explicitly:
curl.config(config);
```

The `baseUrl` config property tells the AMD environment that all module ids
are resolved relative to the given url path.  The path could be absolute
(starts with a protocol or `//`), relative to the host (starts with a `/`),
or relative to the page as show above.

The `paths` config object is a mapping of module ids to urls.  You don't have
to specify every module your app uses.  You can simple specify the top term
in the module's hierarchy and deeper modules will be resolved by appending
their corresponding id  hierarchies.  For instance, here's how a typical
AMD environment might resolve the module "dont/even" given the configuration
above:

* construct a full base url by appending baseUrl to the page location:
	* "//know.cujojs.com/" + "client/apps" => "//know.cujojs.com/client/apps"
* look up "dont/even" in the paths config
	* since it's not found, remove one level and look up "dont"
	* found "dont" which maps to "../../dont"
* resolve the full url by appending "../../dont" to the full base url:
	"//know.cujojs.com/client/apps/../../dont/even"

Although it's possible to map the urls of deep module ids (e.g.
`"dont/even": "../../dont/even"`), you should generally avoid this since it
may not do exactly what you think it does, and it may not work in all AMD
environments.

The `packages` config property also maps ids to urls, but in a more structured
way.  In curl.js, it also provides more advanced features.  Consider using
`packages` instead of `paths` in more sophisticated applications.
