# Creating Reusable Development Tooling

The defining feature of tools like [create-react-app](https://github.com/facebookincubator/create-react-app), my own [nwb](https://github.com/insin/nwb) and the increasing number of [available alternatives](https://github.com/facebookincubator/create-react-app#alternatives) is that they provide an entire development setup managed by a version number.

This is a guide to getting started with creating your own reusable development tooling, with little more overhead than the normal approach of managing development dependencies and configuration in-project.

> The author disclaims responsibility for readers going hog-wild and/or full YAGNI with their development tooling.

## Example application

We need an example application to work with, so let's create a React Hello World project in a fresh directory:

```sh
mkdir src
cat > src/index.js << ^D
```
```js
import React, {Component} from 'react'
import {render} from 'react-dom'

class App extends Component {
  render() {
    return <h1>Hello world!</h1>
  }
}

render(<App/>, document.getElementById('app'))
```
```sh
^D
```

### Install runtime dependencies

```sh
npm init -y
npm install --save react react-dom
```

## Create a separate module for build tooling and config

Instead of using `devDependencies` in your project's own `package.json` and keeping build configuration alongside your application code, create a new module which will own both the development dependencies your need (as its `dependencies`) and the base configuration required to use them.

For the sake of simplicity, start by manually create a new module under `node_modules/`:

```sh
mkdir node_modules/my-scripts/
cd node_modules/my-scripts/
npm init -y
```

Let's set up a production Webpack build for this React app which generates an index.html and minifies the code, with source maps.

The tricks we'll use to do this which make the resulting build configuration independent of our application are:

- using [`require.resolve()`](https://nodejs.org/api/globals.html#globals_require_resolve) to reference JavaScript modules and other files needed for build configuration by absolute paths. We use it below to reference Webpack loaders, Babel presets and an HTML template owned by the `my-scripts` module.

  > This may not work with all tools, for example if you're creating a reusable Karma build config, you may find you need to import plugin modules directly instead.

- using [`path.resolve()`](https://nodejs.org/api/path.html#path_path_resolve_paths) when referencing paths and files related to the application being built, as it resolves paths in the current working directory when given a relative path, which will be the example apps's root directory when running this script.

```sh
npm install --save webpack@3 html-webpack-plugin@2 babel-core@6 babel-loader@7 babel-preset-env@1 babel-preset-react@6
cat > webpack-build.js << ^D
```
```js
let path = require('path')

let HtmlPlugin = require('html-webpack-plugin')
let webpack = require('webpack')

let compiler = webpack({
  devtool: 'source-map',
  entry: {
    app: path.resolve('src/index.js'),
  },
  output: {
    filename: '[name].js',
    path: path.resolve('dist/'),
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: require.resolve('babel-loader'),
        options: {
          presets: [
            require.resolve('babel-preset-env'),
            require.resolve('babel-preset-react'),
          ]
        },
        exclude: /node_modules/,
      }
    ]
  },
  plugins: [
    // Allow development-only code to be eliminated
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify('production'),
    }),
    // Generate an index.html which includes generated scripts
    new HtmlPlugin({
      template: require.resolve('./template.html')
    }),
    // Minify and perform dead code elimination
    new webpack.optimize.UglifyJsPlugin({
      compress: {
        warnings: false,
      },
      output: {
        comments: false,
      },
      sourceMap: true,
    }),
  ],
})

compiler.run((err, stats) => {
  if (err) {
    console.error(err.stack || err)
    if (err.details) {
      console.error(err.details)
    }
    return
  }

  console.log(stats.toString({
    children: false,
    chunks: false,
    colors: true,
  }))
})
```
```sh
^D
cat > template.html << ^D
```
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta http-equiv="x-ua-compatible" content="ie=edge">
    <title>My App</title>
  </head>
  <body>
    <div id="app"></div>
  </body>
</html>
```
```sh
^D
```

## Run the build

Add a run-script to the example app's `package.json`:

```json
{
  "scripts": {
    "build": "node ./node_modules/my-scripts/webpack-build"
  }
}
```

Now go back to the directory your app lives in and run the build:

```sh
$ npm run build

> my-app@1.0.0 build /path/to-my-app
> node ./node_modules/my-scripts/webpack-build

Hash: a07a95da5c66aa112156
Version: webpack 3.8.1
Time: 2375ms
     Asset       Size  Chunks             Chunk Names
    app.js     117 kB       0  [emitted]  app
app.js.map     424 kB       0  [emitted]  app
index.html  349 bytes          [emitted]
...
```

## Follow the same pattern

Use the same approach for any other development tasks you need to set up, such as running a hot reloading development server and running tests, with whichever tools you're using to do so.

This can be done as usual, but using `node_modules/my-scripts/` for development dependencies and configuring paths as we've done above, with the occasional spot of figuring out how to run a new tool this way.

> For example, if you need to use a tool which can't accept Babel config directly (as `babel-loader` for Webpack does above), a quick and dirty solution is to write a temporary `.babelrc` file with the necessary configuration while running.

## Eventually publish

The whole point of using a separate module is that you can eventually publish your build setup to npm or a private registry with a version number, then develop it further as a project in its own right with all the usual benefits a version number brings, like the ability to easily switch between different versions and push out fixes to old versions which are still in use.

If you never create another React project, you might be done with our example `my-scripts` module, but if you need tooling for another project with identical setup and build needs, you can now have it without copy and pasting anything by publishing `my-scripts` and installing it as a `devDependency`.

If you find that certain pieces of configuration need to change while working on a new project, you can make `my-scripts` capable of that, publish the new version and use it.

> There are additional niceties you may want to put in place before publishing, such as creating a [`bin` script](https://docs.npmjs.com/files/package.json#bin) (e.g. we might have configured the build above as `"build": "my-scripts build"` instead with a `bin` script and a bit of plumbing), but the key thing is the version number, which allows you to manage your entire development tooling like any other `devDependency`.

## Keeping up to date

Another benefit of this approach is a vast reduction in the number of `devDependencies` you have to manage in your projects because your development tooling module is handling them for you.

Writing tests for your build tooling will also allow you to take advantage of tools like [Greenkeeper](http://greenkeeper.io/) independent of the projects your tooling is used in, automatically triggering builds when development dependencies are updated, giving you an early warning when there are incompatibilities.

## Playing with shiny new stuff

Having a separate module for development tooling also gives you a place to play with shiny new stuff independent of the projects which are ticking away happily on older versions.

At the time of writing my main project at work is still using Babel 5 and Webpack 1 via an old version of nwb because upgrading build tooling just hasn't been a priority, but when it happens the majority of work in upgrading to Babel 6 and Webpack 3 will be done by bumping a version number.
