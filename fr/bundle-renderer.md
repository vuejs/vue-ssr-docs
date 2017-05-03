# Introducing Bundle Renderer

## Problems with Basic SSR

Up to this point, we have assumed that the bundled server-side code will be directly used by the server via `require`:

``` js
const createApp = require('/path/to/built-server-bundle.js')
```

This is straightforward, however whenever you edit your app source code, you would have to stop and restart the server. This hurts productivity quite a bit during development. In addition, Node.js doesn't support source maps natively.

## Enter BundleRenderer

`vue-server-renderer` provides an API called `createBundleRenderer` to deal with this problem. With a custom webpack plugin, the server bundle is generated as a special JSON file that can be passed to the bundle renderer. Once the bundle renderer is created, usage is the same as the normal renderer, however the bundle renderer provides the following benefits:

- Built-in source map support (with `devtool: 'source-map'` in webpack config)

- Hot-reload during development and even deployment (by simply reading the updated bundle and re-creating the renderer instance)

- Critical CSS injection (when using `*.vue` files): automatically inlines the CSS needed by components used during the render. See the [CSS](./css.md) section for more details.

- Asset injection with [clientManifest](./client-manifest.md): automatically infers the optimal preload and prefetch directives, and the code-split chunks needed for the initial render.

---

We will discuss how to configure webpack to generate the build artifacts needed by the bundle renderer in the next section, but for now let's assume we already have what we need, and this is how to create a use a bundle renderer:

``` js
const { createBundleRenderer } = require('vue-server-renderer')

const renderer = createBundleRenderer(serverBundle, {
  runInNewContext: false, // recommended
  template, // (optinal) page template
  clientManifest // (optional) client build manifest
})

// inside a server handler...
server.get('*', (req, res) => {
  const context = { url: req.url }
  // No need to pass an app here because it is auto-created by the
  // executing the bundle. Now our server is decoupled from our Vue app!
  renderer.renderToString(context, (err, html) => {
    // handle error...
    res.end(html)
  })
})
```

When `rendertoString` is called on a bundle renderer, it will automatically execute the function exported by the bundle to create an app instance (passing `context` as the argument) , and then render it.

---

### The `runInNewContext` Option

By default, for each render the bundle renderer will create a fresh V8 context and re-execute the entire bundle. This has some benefits - for example, we don't need to worry about the "stateful singleton" problem we mentioned earlier. However, this mode comes at some considerable performance cost because re-executing the bundle is expensive especially when the app gets bigger.

In `vue-server-renderer >= 2.3.0`, this option still defaults to `true` for backwards compatibility, but it is recommended to use `runInNewContext: false` whenever you can.
