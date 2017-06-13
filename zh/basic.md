# 基本使用

## 安装

``` bash
npm install vue vue-server-renderer --save
```
教程中使用`NPM`, 但你同样可使用`Yarn`

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

`vue-server-renderer`可以优雅直接的用在`Node.js`服务器中， 比如[Express](https://expressjs.com/)：

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

当渲染Vue应用时， 只会生成应用的HTML标签.  在上面的例子中不得不用HTML页面的壳把输出的标签包起来。

为了简化， 在创建renderer时， 可以提供一个页面模板. 大多数情况会把页面模板放在单独的文件，比如：`index.template.html`:

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
注意`<!--vue-ssr-outlet-->` 注释 -- 这是应用标签注入的地方.

然后就可以读文件内容并传入到Vue renderer中：

``` js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})

renderer.renderToString(app, (err, html) => {
  console.log(html) // will be the full page with app content injected.
})
```

### 模板插值

模板同样支持简单的插值语法， 如下面的模板： 

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
插值数据可以通过 "渲染上下文对象(render context object)" 作为`renderToString`的第二个参数：
  
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


上下文对象能在Vue实例内共享，允许组件为模板插值动态的注册数据。

另外， 模板还支持一些高级的语法：

- Auto injection of critical CSS when using `*.vue` components;
- Auto injection of asset links and resource hints when using `clientManifest`;
- Auto injection and XSS prevention when embedding Vuex state for client-side hydration.

在指南的相关章节我们还会继续讨论这个主题。
