---
layout: post
title: "Using ES7 Decorators with Babel 6"
date: 2015-12-29T10:00:00-05:00
comments: true
author: josh.habdas
categories: [development]
tags: [javascript, transpiling]
cover: /assets/images/covers/broken-bridge.jpg
description: Upgrading to Babel 6 does not mean dropping decorators. Learn to use ES7 decorators with Babel 6.
---

When [Babel 6 dropped](http://babeljs.io/blog/2015/10/29/6.0.0/) on my birthday I was taken aback by all the breaking changes, disclaimers and continued push to use the `.babelrc` file [without a lot of justification](http://babeljs.io/blog/2015/10/29/6.0.0/#comment-2342300088) from the maintainers. Nevertheless I knew Babel 6 was the future, so I went ahead and added support to my [React Native Webpack Starter Kit](https://github.com/jhabdas/react-native-webpack-starter-kit/releases/tag/v1.13.3). Everything went smooth given the small scope of the app, but for some maintainers the Babel 6 upgrade also meant chucking decorators due to [an issue](https://phabricator.babeljs.io/T2645) in Babel 6.

## Add decorator support

But I discovered there's no need to pull your powerful decorators to upgrade to Babel 6. If you'd like restore legacy support for ES7 decorators in Babel 6 do the following:

    npm i -S babel-plugin-transform-decorators-legacy
    
Then update your `.babelrc` file to add the legacy decorator support plug-in like so:

```json
{
  "presets": ["es2015", "react", "stage-1"],
  "plugins": ["babel-plugin-transform-decorators-legacy"]
}
```

If you're using Webpack as a transpiler and module bundler, it's possible to target the plug-in at only desired portions of the codebase by passing the name of the `plugin` to the `query` of a loader, as shown in the example here:

```js
module: {
  loaders: [
    {
      test: /\.js$/,
      exclude: /node_modules\/(?!(stardust))/,
      loader: 'babel',
      query: {
        cacheDirectory: true,
        plugins: [
          'node_modules/babel-plugin-transform-runtime',
          'node_modules/babel-plugin-add-module-exports',
          'node_modules/babel-plugin-transform-decorators-legacy',
        ],
        presets: ['es2015', 'react', 'stage-1'],
      },
    }
  ]
}
```

## Decorator usage

Once added, decorators like [`@autobind`](https://github.com/andreypopp/autobind-decorator) can be used to mitigate the need to override ES6 class constructors, turning this:

```js
class MyClass extends Component {
  constructor(props, context) {
    this.onChange = this.onChange.bind(this)
    this.handleSubmit = this.handleSubmit.bind(this)
    
    this.state = {isLoading: true}
  }
  
  onChange() {}
  handleSubmit() {}
}
```

Into this:

```js
class MyClass extends Component {
  state = {isLoading: true}
  
  @autobind
  onChange() {}
  
  @autobind
  handleSubmit() {}
}
```

Note that Decorators and the `@` operator are an ES7 proposal and are subject to change before the specification is finalized.

## Summary
In this post I've shown you how to add legacy support for Babel 5 ES7 decorators to your Babel 6 apps using Webpack. Decorators are declarative and awesome, and provide a great way to mixin behaviors to your ES6 classes. Got an example usage for Browserify? If so, please do share in the comments section below. Thanks!