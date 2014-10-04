# rework webpack loader

this is a [loader][] for [webpack][] that allows you to create css modules and
process them with [rework][].

it's different than [the other rework webpack loader][other] because it treats
each css file as a different webpack module.

## usage

if you're not familiar with how webpack loaders work, you should checkout [the
main documentation][loaders] first. webpack loaders allow you to transform the
source of webpack modules.

rework allows you to create modular css transformations.

`rework-webpack-loader` works in conjunction with
[`style-loader`][style-loader], but it replaces the standard
[`css-loader`][css-loader].

### installation

```text
$ npm install --save-dev rework-webpack-loader
```

### configuration

you can apply `rework-webpack-loader` to your css either explicitly in the
`require` statements:

```js
require('style!rework-webpack!./index.css');
```

or you can apply it to all css files via a configuration file:

```js
var reworkLoader = require('rework-webpack-loader');
var reworkCalc = require('rework-calc');

module.exports = {
  module: {
    loaders: [
      {test: /\.css$/, loader: 'style-loader!rework-webpack-loader'}
      // ...
    ]
  },

  // the rest of the config...

  rework: {
    use [
      reworkCalc
      // ... other rework plugins ...
    ]
  }
};
```

### advanced configuration example

`rework-webpack-loader` has to do some tricky things to get `@import`'s, url's,
and variables working correctly. so a full featured config might look like
this:

```js
var reworkLoader = require('rework-webpack-loader');
var reworkVars = require('rework-vars');
var reworkCalc = require('rework-calc');
var reworkCustomMedia = require('rework-custom-media');

// if you want to have something like a theme file that can override the css
// variables defined directly in the css file, make a variable map.
var varMap = reworkLoader.makeVarMap('src/index.css');

module.exports = {
  resolve: {
    // if you want to @import stuff you installed with npm, such as suit css,
    // you probably need to include `style` in your `packageMains` config
    packageMains: [
      'webpack',
      'browser',
      'web',
      'browserify',
      ['jam', 'main'],
      'style',
      'main'
    ]
    // ...
  },

  module: {
    loaders: [
      {test: /\.css$/, loader: 'style-loader!rework-webpack-loader'}
      // ...
    ]
  },

  // ...

  rework: {
    use: [
      reworkLoader.plugins.imports,
      reworkLoader.plugins.urls,
      reworkLoader.plugins.stripLocalDefs(varMap),
      reworkCustomMedia({map: varMap}),
      reworkVars({map: varMap}),
      reworkCalc
    ]
  }
};
```

### plugin api

the plugin api is the same as [rework's][reworkplugins], with a couple
additions:

- async plugins: if you retur a promise from a plugin, `rework-webpack-loader`
  will wait for it to complete before applying the next plugin.
- process the resulting JS: `rework-webpack-loader` works similarly to the
  default [`css-loader`][css-loader] in that it transforms css into a
  javascript webpack module that just exports a string of css. in order for
  url's in the css to resolve correctly, the plugin needs to be able to edit
  the resulting js, not just the css, so if plugins define a
  `module.exports.processJs` function, they have the opportunity to transform
  the resulting javascript. check out [the urls plugin][urlplugin] for an
  example.

## motivation

the default `css-loader` plugin wasn't enough for my needs because i wanted
features like variables in my css

the existing `rework-loader` is suboptimal because it doesn't treat each css
file as a separate webpack module, so you lose all of webpack's dependency
tracking a.k.a. the big motivator for using a module system in the first place.

### why are separate modules so important?

to illustrate the importance of treating each css file as a separate module,
consider an app with 3 css files:

- `base.css`: css reset and utility classes
- `menu.css`: styles for a re-usable css drop-down menu component, which
  `@import`'s `base.css` since it uses those utility classes
- `app.css`: styles specific to this application which also `@import`'s
  `base.css` since the app needs the utility classes and reset


while the last file is only ever going to be used for this app, there's a good
chance you'll want to re-use the first two in other apps. so you'll also have a couple javascript modules:

- `menu.js`: a dropdown-menu javascript component which `require`'s `menu.css`
- `app.js`: the rest of your app code, which presumably uses the menu component

so your dependency tree looks something like this:

![dependency tree][deptree]

css preprocessors that don't treat each css file as a separate module will end
up duplicating `base.css` in the final output. this includes the
`rework-loader`, `less-loader`, and most others that trace the `@import`
statements in css for you. while they de-dupe the files internally, since
`menu.css` and `app.css` are different entry points for these preprocessors,
they have no way of knowing about the shared files. when they output the css to
webpack, there is no good way of telling webpack which dependencies are
included in the compiled module.

you also have the development-time annoyance that since webpack doesn't see
these dependencies, making a change to `base.css` won't trigger a
re-compilation.

### why not just chain plugins together?

one approach for solving the dependency issue would be to just compile the
source, but not trace the dependencies in the preprocessor. this could be done
in less by [using a `.css` extension in the `@import` statements](lessimport)
or in rework by not including [the import plugin][reworkimport].

now the problem is resolving variables. css variables are also hard to resolve
at build time since they kind of have 2-way dependencies:

- `base.css` might define a `--base-font` variable and use it to set the
  default font for the app (like in [SUIT CSS][suitbase], for example)
  - this variable is used in `base.css`, so we need to know it's value in order
    to compile the `base.css` module
- `app.css` might reference `--base-font`, so it depends on `base.css`
- `app.css` might also re-define `--base-font`, so now in order to compile
  `base.css`, we need that new value. now `base.css` depends on `app.css`

this is only a problem because we're trying to resolve the values at
build-time, but if we had css variable support in browsers it wouldn't be an
issue ([we don't though][caniusecssvars]).

`rework-webpack-loader` solves this by going through all of the css files up
front and creating a mapping from variable names to values, and then using this
to resolve variables during compilation. the downside is that if you change
variable values, you need to restart the webpack dev server, but the upside is
that it handles css variables' cyclic dependencies.

[loader]: http://webpack.github.io/docs/using-loaders.html
[webpack]: http://webpack.github.io
[rework]: https://github.com/reworkcss/rework
[other]: https://github.com/okonet/rework-loader
[loaders]: http://webpack.github.io/docs/using-loaders.html
[style-loader]: https://github.com/webpack/style-loader
[css-loader]: https://github.com/webpack/css-loader
[suit]: http://suitcss.github.io
[urlplugin]: https://github.com/aaronj1335/rework-webpack-loader/blob/master/lib/plugins/urls.js
[reworkplugins]: https://github.com/reworkcss/rework#plugins
[deptree]: https://i.cloudup.com/_kWP7nyC8_-3000x3000.png
[lessimport]: http://lesscss.org/features/#import-directives-feature
[reworkimport]: https://github.com/reworkcss/rework-import
[suitbase]: https://github.com/suitcss/base/blob/5c8886c4441cdfbf1b28d9ae9810ba907f2a193c/lib/base.css#L4
[caniusecssvars]: http://caniuse.com/#feat=css-variables
