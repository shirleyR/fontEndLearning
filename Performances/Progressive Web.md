### Progressive Web App
原文： 
[A React And Preact Progressive Web App Performance Case Study: Treebo](https://medium.com/dev-channel/treebo-a-react-and-preact-progressive-web-app-performance-case-study-5e4f450d5299)
本文主要介绍Treebo开发者优化网站的过程。Treebo 是印度最大的经济型连锁酒店网站，网站从React切换到Preact在交互时间上有15%的提升。介绍了性能优化之路：


#### 性能优化之旅

#### 旧网站 
#### 单应用React页面
#### Server-side 渲染
他们尝试优化第一次渲染时间，于是采用server-side渲染。这种方法优化了一方面却提高了另一方面的代价。
所谓的服务端渲染是指在后端将完整的HTML页面准备好直接传递给浏览器进行渲染少掉了等待JS的下载和执行时间。

Treebo采用React的renderToString()将组件渲染为一串HTML字符串并且注入到初始的应用中。
```
const serverRenderedHtml = async(req, res, renderPorps)=>{
    const store = configureStore();
    await loadOnServer({...renderProps, store});
    const template = html(
        renderToString(
            <Provider store={store} key="provider">
                <ReduxAsyncConnect {...renderProps} />
            </Provider>
        ),
        store.getState()
    );
    res.send(template);
};

const html = (app, initialState) => '
    <!doctype html>
    <html>
        <head>
            <link rel="stylesheet" href="${assets.main.css}">
        </head>
        <body>
            <div id="root">${app}</div>
            script>window.__INITIAL_STATE__ = ${JSON.stringify(initialState)}</script>
            <script src="${assets.main.js}"></script>
        </body>
    </html>
    </html>
'
```
没看懂例子？？

在Treebo的例子中，使用服务端渲染给他们第一次渲染时间节约了1.1秒，first meaningful较少了2.4秒提高了用户体验。并且改善了SEO测试。但接下来却是消极的影响，影响了用户的交互时间。
虽然用户看到了内容，主线程却是挂接在那，影响了JS。 使用服务端渲染，浏览器相比之前需要去获取和处理大量的HTML载荷，然后解析和运行JS，这实际上是做了更多的工作。第一次的交互时间退化了6.6秒。
服务端渲染在低端设备上带来TTL应答从而锁住主线程

#### 代码分离和基于路由的分块

>Route-based chunking aims to serve the minimal code needed to make a route interactive, by code-splitting the routes into “chunks” that can be loaded on demand. This encourages delivering resources closer to the granularity they were authored in.

Treebo做的事情就是分割他们的依赖。将Webpack runtime manifest和 routes放在不同的快中。

```
// react-middleware.js
//add the webpackManifest and vendor script files to your html

<body>
  <div id="root">${app}</div>
  <script>window.__INITIAL_STATE__ = ${JSON.stringify(initialState)}</script>
  <script src="${assets.webpackManifest.js}"></script>
  <script src="${assets.vendor.js}"></script>
  <script src="${assets.main.js}"></script>
</body>

```
vendor.js
```
import 'redux-pack';
import 'redux-segment';
import 'redux-thunk';
import 'redux';
```
```
// webpack.js

entry: {
  main: './client/index.js',
  vendor: './client/vendor.js',
},
new webpack.optimize.CommonsChunkPlugin({
  names: ['vendor', 'webpackManifest'],
  minChunks: Infinity,
}),


/extract css from all the split chunks into main.hash.css
new ExtractTextPlugin({
  filename: 'css/[name].[contenthash:8].css',
  allChunks: true, 
}),
```

```
// route.js
<Route
  name="landing"
  path="/"
  getComponent={
    (_, cb) => import('./views/LandingPage/LandingPage' /* webpackChunkName: 'landing' */)
      .then((module) => cb(null, module.default))
      .catch((error) => cb(error, null))
  }
</Route>
```

上述的操作将第一次交互时间缩减为4.8秒。

唯一的缺点是,它开始当前路由的JavaScript下载之后才执行完成最初的包,但是从实验的角度，它还是至少产生了积极的效果

对于基于路由的代码分割这种体验，他们正在做一些更隐含的事情。它们使用React Router对getComponent的声明性支持，使用webpack import()调用异步加载数据块。

#### The PRPL Performance Pattern

基于路由分割的代码接绑了细粒度的缓存和服务， Treebo的开发者企图从PRPL模式中找到灵感。
（看到过几篇PRPL的介绍）

PRPL是一种构建模式,注重于应用程序交付和运行的性能。 PRPL代表着 ：
- 为最初的URL路由加载关键资源
- 渲染最初的路由
- 预缓存保留的路由
- 懒加载和按需创建路由

“Push” 部分鼓励对于服务器和浏览器采用非捆绑式的构建设计并且支持HTTP2来为第一次的浏览器渲染所需资源进行缓存。资源的传递可以通过“<link rel="preload">” 或者HTTP/2

Treebo采用的是“<link rel="preload">”预先加载当前路由块。这个带来的影响就是当webpack调用获取初始包完成后执行，降低了他们的交互时间因为当前路由块已经加载缓存好。降低了时间使得第一次交互时间变成4.6秒。

使用preload的唯一缺点就是它不能跨浏览器。然而，在Safari Tech Preview 中实现了link rel preload。 这将是以后的期望。

#### HTML Streaming

renderToString()是一个同步方法，成为了服务端渲染的一个瓶颈。服务端只有在整个HTML发送完成之后才会发送应答。当web服务器流内容相反,浏览器可以渲染页面为用户在整个反应结束之前。react-dom-stream 库可以参考下。

提高性能和引入渐进式渲染应用。Treedo的开发者开始研究HTML streaming。 他们会流head标签link  rel预加载标记给早期预加载CSS和javascript。然后执行服务器端渲染和发送其余的负载到浏览器。
好处是,资源下载开始之前,他们第一次渲染时间降为0.9秒和第一个交互时间变为4.4秒。应用持续交互时间4.9/5秒左右
缺点是它客户端和服务器之间需要保持连接,如果你遇到更长的延迟时间时就会成为问题。
```
earlyChunk(route) {
  return `
    <!doctype html>
    <html lang="en">
      <head>
        <link rel="stylesheet" href="${assets.main.css}">
        <link rel="preload" as="script" href="${assets.webpackManifest.js}">
        <link rel="preload" as="script" href="${assets.vendor.js}">
        <link rel="preload" as="script" href="${assets.main.js}">
        ${!assets[route.name] ? '' : `<link rel="preload" as="script" href="${assets[route.name].js}">`}
      </head>`;
},
lateChunk(app, head, initialState) {
  return `
      <body>
        <div id="root">${app}</div>
        <script>window.__INITIAL_STATE__ = ${JSON.stringify(initialState)}</script>
        <script src="${assets.webpackManifest.js}"></script>
        <script src="${assets.vendor.js}"></script>
        <script src="${assets.main.js}"></script>
      </body>
    </html>
  `;
},
```
```
//react-middleware.js

const serverRenderedChunks = async (req, res, renderProps) => {
  const route = renderProps.routes[renderProps.routes.length - 1];
  const store = configureStore();
  //set the content type since you're streaming the response
  res.set('Content-Type', 'text/html');
  //flush the head with css & js resource tags first so the download starts immediately
  const earlyChunk = html.earlyChunk(route);
  res.write(earlyChunk);
  res.flush();
  //call & wait for api's response, set them into state
  await loadOnServer({ ...renderProps, store });
  //flush the rest of the body once app the server side rendered
  const lateChunk = html.lateChunk(
    renderToString(
      <Provider store={store} key="provider">
        <ReduxAsyncConnect {...renderProps} />
      </Provider>,
    ),
    Helmet.renderStatic(),
    store.getState(),
    route,
  );
  res.write(lateChunk);
  res.flush();
  //let client know the response has ended
  res.end();
};
```

未完。。。



