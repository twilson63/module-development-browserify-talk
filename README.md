title: Modular Client Development using NodeJS Browserify
author:
  name: Tom Wilson
  twitter: twilson63
  url: http://twilson63.com
output: index.html
controls: true

--

# Modular Client Development
## using NodeJS - Browserify

--

## What is a modular application?

--

The concept of modular applications, is breaking up your code in small re-usable pieces, separting the custom from 
the generic to maximize the ability to reuse focusing on
the `single responsibility principal` - [https://en.wikipedia.org/wiki/Single_responsibility_principle](https://en.wikipedia.org/wiki/Single_responsibility_principle)

--

# What is Browserify?

--

Browserify is a node application that uses the node-flavored module system and some static analysis to bundle up your javascript modules into a single file that can be served to the client.

But before we jump in to Browserify, it is important to understand how modules work in nodejs.

--

## What is NodeJS?

--

* Server-Side
* NonBlocking IO
* JavaScript on the Server
* Open/Diverse Community

--

# What is `node-flavored` commonjs?

--

### require

In node, there is a `require()` function for loading code in other files.

--

### Lets install a module with npm 

```
npm install uniq
```

--

### Then lets create a file called `num.js`, and we can `require('uniq')`

``` 
var uniq = require('uniq')
var nums = [ 5, 2, 1, 3, 2, 5, 4, 2, 0, 1 ]
console.log(uniq(nums))
```

--

### Output

``` 
node nums.js
[ 0, 1, 2, 3, 4, 5 ]
```

--

### You can also require relative files to your file.

``` 
var foo = require('./foo.js')
console.log(foo(4))
```

--

### if `foo` was in the parent directory, you could use `..foo.js` instead

``` 
var foo = require('../foo.js');
console.log(foo(4));
```

--

How `require()` works is unlike many other module systems where imports are akin to statements that expose themselves as globals or file-local lexicals with names declared in the module itself outside of your control. 

Under the node style of code import with `require()`, someone reading your program can easily tell where each piece of functionality came from. This approach scales much better as the number of modules in an application grows.

--

# exports

--

To export a single thing from a file so that other files may import it, assign over the value at module.exports:

```
module.exports = function (n) {
    return n * 111
};

```

--

Now when some module main.js loads your foo.js, the return value of require('./foo.js') will be the exported function:

```
var foo = require('./foo.js');
console.log(foo(5));
```

This program will print:

```
555
```

--

You can export any kind of value with module.exports, not just functions.

For example, this is perfectly fine:

```
module.exports = 555
```

and so is this:

```
var numbers = [];
for (var i = 0; i < 100; i++) numbers.push(i);

module.exports = numbers;
```

--

There is another form of doing exports specifically for exporting items onto an object. Here, exports is used instead of module.exports:

```
exports.beep = function (n) { return n * 1000 }
exports.boop = 555
```

This program is the same as:

```
module.exports.beep = function (n) { return n * 1000 }
module.exports.boop = 555
```

because module.exports is the same as exports and is initially set to an empty object.

--

Note however that you can't do:

```
// this doesn't work
exports = function (n) { return n * 1000 }
```

because the export value lives on the module object, and so assigning a new value for exports instead of module.exports masks the original reference.

Instead if you are going to export a single item, always do:

```
// instead
module.exports = function (n) { return n * 1000 }
```

--

If you're still confused, try to understand how modules work in the background:

```
var module = {
  exports: {}
};

// If you require a module, it's basically wrapped in a function
(function(module, exports) {
  exports = function (n) { return n * 1000 };
}(module, module.exports))

console.log(module.exports); // it's still an empty object :(
```

--

Most of the time, you will want to export a single function or constructor with module.exports because it's usually best for a module to do one thing.

The exports feature was originally the primary way of exporting functionality and module.exports was an afterthought, but module.exports proved to be much more useful in practice at being more direct, clear, and avoiding duplication.

--

In the early days, this style used to be much more common:

foo.js:

```
exports.foo = function (n) { return n * 111 }
```

main.js:

```
var foo = require('./foo.js');
console.log(foo.foo(5));
```

but note that the foo.foo is a bit superfluous.

--

Using module.exports it becomes more clear:

foo.js:

```
module.exports = function (n) { return n * 111 }
```

main.js:

```
var foo = require('./foo.js');
console.log(foo(5));
```

--

# Bundling for the browser

--

To run a module in node, you've got to start from somewhere.

In node you pass a file to the node command to run a file:

```
node robot.js
beep boop
```

--

In browserify, you do this same thing, but instead of running the file, you generate a stream of concatenated javascript files on stdout that you can write to a file with the > operator:

```
browserify robot.js > bundle.js
```

--

Now `bundle.js` contains all the javascript that robot.js needs to work. Just plop it into a single script tag in some html:

```
<html>
  <body>
    <script src="bundle.js"></script>
  </body>
</html>
```

Bonus: if you put your script tag right before the </body>, you can use all of the dom elements on the page without waiting for a dom onready event.

--

# How browserify works

--

Browserify starts at the entry point files that you give it and searches for any require() calls it finds using static analysis of the source code's abstract syntax tree.

For every require() call with a string in it, browserify resolves those module strings to file paths and then searches those file paths for require() calls recursively until the entire dependency graph is visited.

Each file is concatenated into a single javascript file with a minimal require() definition that maps the statically-resolved names to internal IDs.

--

This means that the bundle you generate is completely self-contained and has everything your application needs to work with a pretty negligible overhead.

For more details about how browserify works, check out the compiler pipeline section of this document.

--

# how node_modules work

--

node has a clever algorithm for resolving modules that is unique among rival platforms.

Instead of resolving packages from an array of system search paths like how $PATH works on the command line, node's mechanism is local by default.

If you `require('./foo.js')` from `/beep/boop/bar.js`, node will look for `./foo.js` in `/beep/boop/foo.js`. Paths that start with a `./` or `../` are always local to the file that calls `require()`.

--

If however you require a non-relative name such as require('xyz') from /beep/boop/foo.js, node searches these paths in order, stopping at the first match and raising an error if nothing is found:

```
/beep/boop/node_modules/xyz
/beep/node_modules/xyz
/node_modules/xyz
```

For each xyz directory that exists, node will first look for a xyz/package.json to see if a "main" field exists. The "main" field defines which file should take charge if you require() the directory path.

--

For example, if `/beep/node_modules/xyz` is the first match and `/beep/node_modules/xyz/package.json` has:

```
{
  "name": "xyz",
  "version": "1.2.3",
  "main": "lib/abc.js"
}
```

then the exports from `/beep/node_modules/xyz/lib/abc.js` will be returned by `require('xyz')`.

--

If there is no `package.json` or no "main" field, `index.js` is assumed:

```
/beep/node_modules/xyz/index.js
```

--

If you need to, you can reach into a package to pick out a particular file. For example, to load the `lib/clone.js` file from the dat package, just do:

```
var clone = require('dat/lib/clone.js')
```

The recursive `node_modules` resolution will find the first dat package up the directory hierarchy, then the lib/clone.js file will be resolved from there. This `require('dat/lib/clone.js')` approach will work from any location where you can `require('dat')`.

--

Each library gets its own local node_modules/ directory where its dependencies are stored and each dependency's dependencies has its own node_modules/ directory, recursively all the way down.

This means that packages can successfully use different versions of libraries in the same application, which greatly decreases the coordination overhead necessary to iterate on APIs. This feature is very important for an ecosystem like npm where there is no central authority to manage how packages are published and organized. Everyone may simply publish as they see fit and not worry about how their dependency version choices might impact other dependencies included in the same application.

You can leverage how node_modules/ works to organize your own local application modules too. See the avoiding `../../../../../../..` section for more.

--

# why concatenate

--

Browserify is a build step that runs on the server. It generates a single bundle file that has everything in it.

Here are some other ways of implementing module systems for the browser and what their strengths and weaknesses are:

--

### window globals

Instead of a module system, each file defines properties on the window global object or develops an internal namespacing scheme.

This approach does not scale well without extreme diligence since each new file needs an additional `<script>` tag in all of the html pages where the application will be rendered. Further, the files tend to be very order-sensitive because some files need to be included before other files the expect globals to already be present in the environment.

It can be difficult to refactor or maintain applications built this way. On the plus side, all browsers natively support this approach and no server-side tooling is required.

This approach tends to be very slow since each `<script>`tag initiates a new round-trip http request.

--

### concatenate

Instead of window globals, all the scripts are concatenated beforehand on the server. The code is still order-sensitive and difficult to maintain, but loads much faster because only a single http request for a single `<script>` tag needs to execute.

Without source maps, exceptions thrown will have offsets that can't be easily mapped back to their original files.

--

### AMD

Instead of using `<script>` tags, every file is wrapped with a define() function and callback. This is AMD.

The first argument is an array of modules to load that maps to each argument supplied to the callback. Once all the modules are loaded, the callback fires.

```
define(['jquery'] , function ($) {
    return function () {};
});
```

You can give your module a name in the first argument so that other modules can include it.

--

There is a commonjs sugar syntax that stringifies each callback and scans it for `require()` calls with a regexp.

Code written this way is much less order-sensitive than concatenation or globals since the order is resolved by explicit dependency information.

For performance reasons, most of the time AMD is bundled server-side into a single file and during development it is more common to actually use the asynchronous feature of AMD.

--

### bundling commonjs server-side 

If you're going to have a build step for performance and a sugar syntax for convenience, why not scrap the whole AMD business altogether and bundle commonjs? With tooling you can resolve modules to address order-sensitivity and your development and production environments will be much more similar and less fragile. The CJS syntax is nicer and the ecosystem is exploding because of node and npm.

You can seamlessly share code between node and the browser. You just need a build step and some tooling for source maps and auto-rebuilding.

Plus, we can use node's module lookup algorithms to save us from version mismatch insanity so that we can have multiple conflicting versions of different required packages in the same application and everything will still work. To save bytes down the wire you can dedupe, which is covered elsewhere in this document.

--

### Development

Concatenation has some downsides, but these can be very adequately addressed with development tooling.

--

### source maps

Browserify supports a --debug/-d flag and opts.debug parameter to enable source maps. Source maps tell the browser to convert line and column offsets for exceptions thrown in the bundle file back into the offsets and filenames of the original sources.

The source maps include all the original file contents inline so that you can simply put the bundle file on a web server and not need to ensure that all the original source contents are accessible from the web server with paths set up correctly.

--

### exorcist

The downside of inlining all the source files into the inline source map is that the bundle is twice as large. This is fine for debugging locally but not practical for shipping source maps to production. However, you can use exorcist to pull the inline source map out into a separate bundle.map.js file:

```
browserify main.js --debug | exorcist bundle.js.map > bundle.js
```

--

### auto-recompile

Running a command to recompile your bundle every time can be slow and tedious. Luckily there are many tools to solve this problem. Some of these tools support live-reloading to various degrees and others have a more traditional manual refresh cycle.

--

### watchify

You can use `watchify` interchangeably with `browserify` but instead of writing to an output file once, watchify will write the bundle file and then watch all of the files in your dependency graph for changes. When you modify a file, the new bundle file will be written much more quickly than the first time because of aggressive caching.

--

Here is a handy configuration for using watchify and browserify with the package.json "scripts" field:

```
{
  "build": "browserify browser.js -o static/bundle.js",
  "watch": "watchify browser.js -o static/bundle.js --debug --verbose",
}
```

--

### wzrd

In a similar spirit to beefy but in a more minimal form is wzrd.

Just `npm install -g wzrd` then you can do:

```
wzrd app.js
```

and open up http://localhost:9966 in your browser.

--

# transforms

--

Instead of browserify baking in support for everything, it supports a flexible transform system that are used to convert source files in-place.

This way you can require() files written in coffee script or templates and everything will be compiled down to javascript.

--

To use coffeescript for example, you can use the coffeeify transform. Make sure you've installed coffeeify first with npm install coffeeify then do:

```
browserify -t coffeeify main.coffee > bundle.js
```

or with the API you can do:

```
var b = browserify('main.coffee');
b.transform('coffeeify');
```

--

The best part is, if you have source maps enabled with --debug or opts.debug, the bundle.js will map exceptions back into the original coffee script source files. This is very handy for debugging with firebug or chrome inspector.

--

# usage - tutorial

```
npm install -g browserify-adventure browserify
browserify-adventure
```

--

### Credits

All credit goes to `https://github.com/substack/browserify-handbook` and `substack`

This presentation is just putting the handbook in a slideshow format.


