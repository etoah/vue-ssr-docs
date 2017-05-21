# 基本使用

## 安装

``` bash
npm install vue vue-server-renderer --save
```
教程中使用`NPM`, 但你也可使用`Yarn`

#### 注意
- 推荐使用 Node.js 6+.
- `vue-server-renderer` 和 `vue` 版本必须匹配.
- `vue-server-renderer` 依赖于Node.js原生的模块，因此只能用于Node.js 运行时. 我们在将来会提供一个包可以运行在其它的运行时.


## 渲染Vue实例

``` js
// Step 1: 创建Vue实例
const Vue = require('vue')
const app = new Vue({
  template: `<div>Hello World</div>`
})

// Step 2: 创建renderer
const renderer = require('vue-server-renderer').createRenderer()

// Step 3: 把实例渲染成HTML
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <div data-server-rendered="true">hello world</div>
})
```

## 和服务器整合

It is pretty straightforward when used inside a Node.js server, for example [Express](https://expressjs.com/):

``` bash
npm install express --save
```
---
``` js
const Vue = require('vue')
const server = require('express')()
const renderer = require('vue-server-renderer').createRenderer()

server.get('*', (req, res) => {
  const app = new Vue({
    data: {
      url: req.url
    },
    template: `<div>The visited URL is: {{ url }}</div>`
  })

  renderer.renderToString(app, (err, html) => {
    if (err) {
      res.status(500).end('Internal Server Error')
      return
    }
    res.end(`
      <!DOCTYPE html>
      <html lang="en">
        <head><title>Hello</title></head>
        <body>${html}</body>
      </html>
    `)
  })
})

server.listen(8080)
```

## 用页面模板

When you render a Vue app, the renderer only generates the markup of the app. In the example we had to wrap the output with an extra HTML page shell.

To simplify this, you can directly provide a page template when creating the renderer. Most of the time we will put the page template in its own file, e.g. `index.template.html`:

``` html
<!DOCTYPE html>
<html lang="en">
  <head><title>Hello</title></head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

Notice the `<!--vue-ssr-outlet-->` comment -- this is where your app's markup will be injected.

We can then read and pass the file to the Vue renderer:

``` js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})

renderer.renderToString(app, (err, html) => {
  console.log(html) // will be the full page with app content injected.
})
```

### 模板插值

The template also supports simple interpolation. Given the following template:

``` html
<html>
  <head>
    <!-- use double mustache for HTML-escaped interpolation -->
    <title>{{ title }}</title>
    
    <!-- use triple mustache for non-HTML-escaped interpolation -->
    {{{ meta }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

We can provide interpolation data by passing a "render context object" as the second argument to `renderToString`:

``` js
const context = {
  title: 'hello',
  meta: `
    <meta ...>
    <meta ...>
  `
}

renderer.renderToString(app, context, (err, html) => {
  // page title will be "hello"
  // with meta tags injected
})
```

The `context` object can also be shared with the Vue app instance, allowing components to dynamically register data for template interpolation.

In addition, the template supports some advanced features such as:

- Auto injection of critical CSS when using `*.vue` components;
- Auto injection of asset links and resource hints when using `clientManifest`;
- Auto injection and XSS prevention when embedding Vuex state for client-side hydration.

We will discuss these when we introduce the associated concepts later in the guide.
