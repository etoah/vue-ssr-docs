# API 指南

## `createRenderer([options])`

Create a [`Renderer`](#class-renderer) instance with (optional) [options](#renderer-options).

``` js
const { createRenderer } = require('vue-server-renderer')
const renderer = createRenderer({ ... })
```

## `createBundleRenderer(bundle[, options])`

Create a [`BundleRenderer`](#class-bundlerenderer) instance with a server bundle and (optional) [options](#renderer-options).

``` js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, { ... })
```

The `serverBundle` argument can be one of the following:

- An absolute path to generated bundle file (`.js` or `.json`). Must start with `/` to be treated as a file path.

- A bundle object generated by webpack + `vue-server-renderer/server-plugin`.

- A string of JavaScript code (not recommended).

See [Introducing the Server Bundle](./bundle-renderer.md) and [Build Configuration](./build-config.md) for more details.

## `Class: Renderer`

- #### `renderer.renderToString(vm[, context], callback)`

  Render a Vue instance to string. The context object is optional. The callback is a typical Node.js style callback where the first argument is the error and the second argument is the rendered string.

- #### `renderer.renderToStream(vm[, context])`

  Render a Vue instance to a Node.js stream. The context object is optional. See [Streaming](./streaming.md) for more details.

## `Class: BundleRenderer`

- #### `bundleRenderer.renderToString([context, ]callback)`

  Render the bundle to a string. The context object is optional. The callback is a typical Node.js style callback where the first argument is the error and the second argument is the rendered string.

- #### `bundleRenderer.renderToStream([context])`

  Render the bundle to a Node.js stream. The context object is optional. See [Streaming](./streaming.md) for more details.

## Renderer Options

- #### `template`

  Provide a template for the entire page's HTML. The template should contain a comment `<!--vue-ssr-outlet-->` which serves as the placeholder for rendered app content.

  The template also supports basic interpolation using the render context:

  - Use double-mustache for HTML-escaped interpolation;
  - Use triple-mustache for non-HTML-escaped interpolation.

  The template automatically injects appropriate content when certain data is found on the render context:

  - `context.head`: (string) any head markup that should be injected into the head of the page.

  - `context.styles`: (string) any inline CSS that should be injected into the head of the page. Note this property will be automatically populated if using `vue-loader` + `vue-style-loader` for component CSS.

  - `context.state`: (Object) initial Vuex store state that should be inlined in the page as `window.__INITIAL_STATE__`. The inlined JSON is automatically sanitized with [serialize-javascript](https://github.com/yahoo/serialize-javascript) to prevent XSS.

  In addition, when `clientManifest` is also provided, the template automatically injects the following:

  - Client-side JavaScript and CSS assets needed by the render (with async chunks automatically inferred);
  - Optimal `<link rel="preload/prefetch">` resource hints for the rendered page.

  You can disable all automatic injections by also passing `inject: false` to the renderer.

  See also:

  - [Using a Page Template](./basic.md#using-a-page-template)
  - [Manual Asset Injection](./build-config.md#manual-asset-injection)

- #### `clientManifest`

  - 2.3.0+

  Provide a client build manifest object generated by `vue-server-renderer/client-plugin`. The client manifest provides the bundle renderer with the proper information for automatic asset injection into the HTML template. For more details, see [Generating clientManifest](./build-config.md#generating-clientmanifest).

- #### `inject`

  - 2.3.0+

  Controls whether to perform automatic injections when using `template`. Defaults to `true`.

  See also: [Manual Asset Injection](./build-config.md#manual-asset-injection).

- #### `shouldPreload`

  - 2.3.0+

  A function to control what files should have `<link rel="preload">` resource hints generated.

  By default, only JavaScript and CSS files will be preloaded, as they are absolutely needed for your application to boot.

  For other types of assets such as images or fonts, preloading too much may waste bandwidth and even hurt performance, so what to preload will be scenario-dependent. You can control precisely what to preload using the `shouldPreload` option:

  ``` js
  const renderer = createBundleRenderer(bundle, {
    template,
    clientManifest,
    shouldPreload: (file, type) => {
      // type is inferred based on the file extension.
      // https://fetch.spec.whatwg.org/#concept-request-destination
      if (type === 'script' || type === 'style') {
        return true
      }
      if (type === 'font') {
        // only preload woff2 fonts
        return /\.woff2$/.test(file)
      }
      if (type === 'image') {
        // only preload important images
        return file === 'hero.jpg'
      }
    }
  })
  ```

- #### `runInNewContext`

  - 2.3.0+
  - only used in `createBundleRenderer`
  - Expects: `boolean | 'once'` (`'once'` only supported in 2.3.1+)

  By default, for each render the bundle renderer will create a fresh V8 context and re-execute the entire bundle. This has some benefits - for example, the app code is isolated from the server process and we don't need to worry about the [stateful singleton problem](./structure.md#avoid-stateful-singletons) mentioned in the docs. However, this mode comes at some considerable performance cost because re-executing the bundle is expensive especially when the app gets bigger.

  This option defaults to `true` for backwards compatibility, but it is recommended to use `runInNewContext: false` or `runInNewContext: 'once'` whenever you can.

  > In 2.3.0 this option has a bug where `runInNewContext: false` still executes the bundle using a separate global context. The following information assumes version 2.3.1+.

  With `runInNewContext: false`, the bundle code will run in the same `global` context with the server process, so be careful about code that modifies `global` in your application code.

  With `runInNewContext: 'once'` (2.3.1+), the bundle is evaluated in a separate `global` context, however only once at startup. This provides better app code isolation since it prevents the bundle from accidentally polluting the server process' `global` object. The caveats are that:

  1. Dependencies that modifies `global` (e.g. polyfills) cannot be externalized in this mode;
  2. Values returned from the bundle execution will be using different global constructors, e.g. an error caught inside the bundle will not be an instance of `Error` in the server process.

  See also: [Source Code Structure](./structure.md)

- #### `basedir`

  - 2.2.0+
  - only used in `createBundleRenderer`

  Explicitly declare the base directory for the server bundle to resolve `node_modules` dependencies from. This is only needed if your generated bundle file is placed in a different location from where the externalized NPM dependencies are installed, or your `vue-server-renderer` is npm-linked into your current project.

- #### `cache`

  Provide a [component cache](./caching.md#component-level-caching) implementation. The cache object must implement the following interface (using Flow notations):

  ``` js
  type RenderCache = {
    get: (key: string, cb?: Function) => string | void;
    set: (key: string, val: string) => void;
    has?: (key: string, cb?: Function) => boolean | void;
  };
  ```

  A typical usage is passing in an [lru-cache](https://github.com/isaacs/node-lru-cache):

  ``` js
  const LRU = require('lru-cache')

  const renderer = createRenderer({
    cache: LRU({
      max: 10000
    })
  })
  ```

  Note that the cache object should at least implement `get` and `set`. In addition, `get` and `has` can be optionally async if they accept a second argument as callback. This allows the cache to make use of async APIs, e.g. a redis client:

  ``` js
  const renderer = createRenderer({
    cache: {
      get: (key, cb) => {
        redisClient.get(key, (err, res) => {
          // handle error if any
          cb(res)
        })
      },
      set: (key, val) => {
        redisClient.set(key, val)
      }
    }
  })
  ```

- #### `directives`

  Allows you to provide server-side implementations for your custom directives:

  ``` js
  const renderer = createRenderer({
    directives: {
      example (vnode, directiveMeta) {
        // transform vnode based on directive binding metadata
      }
    }
  })
  ```

  As an example, check out [`v-show`'s server-side implementation](https://github.com/vuejs/vue/blob/dev/src/platforms/web/server/directives/show.js).

## Webpack Plugins

The webpack plugins are provided as standalone files and should be required directly:

``` js
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
```

The default files generated are:

- `vue-ssr-server-bundle.json` for the server plugin;
- `vue-ssr-client-manifest.json` for the client plugin.

The filenames can be customized when creating the plugin instances:

``` js
const plugin = new VueSSRServerPlugin({
  filename: 'my-server-bundle.json'
})
```

详情请查看[构建配置](./build-config.md).
