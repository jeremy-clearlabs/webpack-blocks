# webpack-blocks

Functional building blocks for the webpack config. Compose it using feature middlewares like *Babel*, *PostCSS*, *HMR&nbsp;(Hot&nbsp;Module&nbsp;Replacement)*, …

Divide & conquer your configuration - Cluster your blocks to groups and compose these groups.

Missing anything? Write your own blocks and share them!

[![Build Status](https://travis-ci.org/andywer/webpack-blocks.svg?branch=master)](https://travis-ci.org/andywer/webpack-blocks)
[![JavaScript Style Guide](https://img.shields.io/badge/code%20style-standard-brightgreen.svg)](http://standardjs.com/)

>"Finally, webpack config done right. (...) Webpack clearly wants to stay low-level. So it makes total sense to outsource configuring it to well designed blocks instead of copy-paste."
>
>[Dan Abramov](https://github.com/gaearon) via [twitter](https://twitter.com/dan_abramov/status/806249934399881216) (Co-author of Redux, Create React App and React Hot Loader)


## Table of Contents

- [Installation](#installation)
- [Usage](#usage)
- [Available webpack blocks](#available-webpack-blocks)
- [Design principles](#design-principles)
- [FAQ](#faq)
- [Like what you see?](#like-what-you-see)
- [License](#license)


## Installation

```sh
npm install --save-dev webpack webpack-blocks
# or
yarn add --dev webpack webpack-blocks
```


## Usage

The following sample shows how to create a webpack config with Babel support, dev server and PostCSS autoprefixer.

All blocks, like `babel6`, `postcss` and so on, live in their own small packages. We added the unscoped [webpack-blocks package](./packages/webpack-blocks) that wraps all the most common blocks, so you do not have to manage these micro dependencies.

```js
const {
  createConfig,
  match,
  webpack,

  // Feature blocks
  babel,
  css,
  devServer,
  file,
  postcss,

  // Shorthand setters
  addPlugins,
  defineConstants,
  entryPoint,
  env,
  setOutput,
  sourceMaps
} = require('webpack-blocks')
const autoprefixer = require('autoprefixer')
const path = require('path')

module.exports = createConfig([
  entryPoint('./src/main.js'),
  setOutput('./build/bundle.js'),
  babel(),
  match('*.css', { exclude: path.resolve('node_modules') }, [
    css(),
    postcss([
      autoprefixer({ browsers: ['last 2 versions'] })
    ])
  ]),
  match(['*.gif', '*.jpg', '*.jpeg', '*.png', '*.webp'], [
    file()
  ]),
  defineConstants({
    'process.env.NODE_ENV': process.env.NODE_ENV
  }),
  env('development', [
    devServer(),
    devServer.proxy({
      '/api': { target: 'http://localhost:3000' }
    }),
    sourceMaps()
  ]),
  env('production', [
    addPlugins([
      new webpack.LoaderOptionsPlugin({ minimize: true })
    ])
  ])
])
```

Wanna use CSS modules? No problem!

```js
const { css } = require('webpack-blocks')

...

module.exports = createConfig([
  ...
  match('*.css', { exclude: path.resolve('node_modules') }, [
    css.modules()
  ]
])
```

Need a custom block? A block is quite easy to write:

```js
module.exports = createConfig([
  ...
  myCssLoader([ './styles' ])
])

function myCssLoader () {
  return (context, { merge }) => merge({
    module: {
      rules: [
        Object.assign(
          {
            test: context.fileType('text/css'),
            use: [ 'style-loader', 'my-css-loader' ]
          },
          context.match     // carries `test`, `exclude` & `include` as set by `match()`
        )
      ]
    }
  })
}
```

If we use `myCssLoader` in `match()` then `context.match` will be populated with whatever we set in `match()`. Otherwise there is still the `test: context.fileType('text/css')` fallback, so our block will work without `match()` as well.

Check out the [sample app](./test-app) to see a webpack config in action or read [how to create your own blocks](./docs/BLOCK-CREATION.md).


## Available webpack blocks

- [assets](./packages/assets)
- [babel6](./packages/babel6)
- [dev-server](./packages/dev-server)
- [extract-text](./packages/extract-text)
- [postcss](./packages/postcss)
- [sass](./packages/sass)
- [typescript](./packages/typescript)
- [tslint](./packages/tslint)
- [webpack](./packages/webpack)

Missing something? Write and publish your own webpack blocks!

## Design principles

- Extensibility first
- Uniformity for easy composition
- Keep everything configurable
- But provide sane defaults

## FAQ

<details>
<summary>How to debug?</summary>

In case the webpack configuration does not work as expected you can easily debug it using [stringify-object](https://www.npmjs.com/package/stringify-object):

```js
const stringify = require('stringify-object')

module.exports = createConfig([
  // ...
])

console.log(stringify(module.exports))
```
</details>

<details>
<summary>How does env() work?</summary>

You might wonder how `env('development', [ ... ])` works? It just checks the NODE_ENV environment variable and only applies its contained webpack blocks if it matches.

So make sure you set the NODE_ENV accordingly:

```js
// your package.json
"scripts": {
  "build": "NODE_ENV=production webpack --config webpack.config.js",
  "start": "NODE_ENV=development webpack-dev-server --config webpack.config.js"
}
```

If there is no NODE_ENV set then it will just treat NODE_ENV as if it was `development`.
</details>

<details>
<summary>What does defineConstants() do?</summary>

`defineConstants()` is just a small convenience wrapper around webpack's [DefinePlugin](https://webpack.github.io/docs/list-of-plugins.html#defineplugin). It is composable and automatically encodes the values. Use it to replace constants in your code by their values at build time.

So having a `defineConstants({ 'process.env.FOO': 'foo' })` and a `defineConstants({ 'process.env.BAR': 'bar' })` in your config means the resulting webpack config will finally contain a single `new webpack.DefinePlugin({ 'process.env.FOO': '"FOO"', 'process.env.BAR': '"BAR"' })`, thus replacing any occurence of `process.env.FOO` and `process.env.BAR` with the given values.
</details>

<details>
<summary>What does a block look like from the inside?</summary>

A webpack block is *just a function and requires no dependencies at all* (🎉🎉), thus making it easy to write your own blocks and share them with the community.

Take the `babel6` webpack block for instance:

```js
/**
 * @param {object} [options]
 * @param {RegExp|Function|string}  [options.exclude]   Directories to exclude.
 * @return {Function}
 */
function babel (options = { cacheDirectory: true }) {
  return (context, util) => util.addLoader(
    Object.assign({
      // we use a `MIME type => RegExp` abstraction here in order to have consistent regexs
      test: context.fileType('application/javascript'),
      exclude: /node_modules/,
      use: [
        { loader: 'babel-loader', options }
      ]
    }, context.match)
  )
}
```

Add a README and a package.json and you are ready to ship.

For more details see [How to write a block](./docs/BLOCK-CREATION.md).
</details>

<details>
<summary>I need some custom webpack config snippet!</summary>

No problem. If you don't want to write your own webpack block you can just use `customConfig()`:

```js
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const { addPlugins, customConfig } = require('@webpack-blocks/webpack')

...

module.exports = createConfig([
  ...
  addPlugins([
    // Add a custom webpack plugin
    new HtmlWebpackPlugin({
      inject: true,
      template: './index.html'
    })
  ]),
  customConfig({
    // Add some custom webpack config snippet
    resolve: {
      extensions: [ '.js', '.es6' ]
    }
  })
])
```

The object you pass to `customConfig()` will be merged into the webpack config using
[webpack-merge](https://github.com/survivejs/webpack-merge) like any other webpack
block's partial config.
</details>

<details>
<summary>How to compose blocks? (a.k.a. building presets)</summary>

Got some projects with a similar, yet not identical webpack configuration? Seems like you could use a preset:

```js
const { createConfig, env, group } = require('@webpack-blocks/webpack')
const babel = require('@webpack-blocks/babel6')
const devServer = require('@webpack-blocks/dev-server')

function myPreset (proxyConfig) {
  return group([
    babel(),
    env('development', [
      devServer(),
      devServer.proxy(proxyConfig)
    ])
  ])
}

module.exports = createConfig([
  myPreset({
    '/api': { target: 'http://localhost:3000' }
  }),
  ...   // add more blocks here
])
```

The key feature is the `group()` method which takes a set of blocks and returns a new block that combines all their functionality.
</details>


## Like what you see?

Support webpack-blocks by giving [feedback](https://github.com/andywer/webpack-blocks/issues), publishing new webpack blocks or just by 🌟 starring the project!


## License

MIT
