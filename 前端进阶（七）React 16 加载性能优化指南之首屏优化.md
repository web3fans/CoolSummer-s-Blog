https://cloud.tencent.com/developer/article/1358160  React 16 加载性能优化指南（上）

https://cloud.tencent.com/developer/article/1359124  React 16 加载性能优化指南（下）

这篇文章是针对这两篇文章的笔记，只为总结React 16 加载性能优化。

------------------------------------------------------------------------------------------------

##一. 打开页面 -> 首屏（下载正式html+js+css文件）

1. 在 root 节点中写一些东西,比如Loading...或者UI单独设计一个统一的样式。

2. 使用 html-webpack-plugin 自动插入 loading,运行时同步加载一个写好的页面（loading html文件和css文件），等到正式页面文件下载完成后替换掉

3. 使用 prerender-spa-plugin 渲染首屏

4. 除掉外链 css

##二. 首屏 -> 首次内容渲染（加载、运行 JS 代码）

几乎所有业务的 JS 代码，都可以大致划分成以下几个大块：

1. 基础框架，如 React、Vue 等，这些基础框架的代码是不变的，除非升级框架；

2. Polyfill，对于使用了 ES2015+ 语法的项目来说，为了兼容性，polyfill 是必要的存在；

3. 业务基础库，业务的一些通用的基础代码，不属于框架，但大部分业务都会使用到；

4. 业务代码，特点是具体业务自身的逻辑代码。

下面分别详细讲解：

####1.缓存基础框架

特点是必须且不变，为基础框架代码设置一个尽量长的缓存时间，使用户的浏览器尽量通过缓存加载这些资源。

附：HTTP 缓存资源小结

HTTP 为我们提供了很好几种缓存的解决方案，不妨总结一下：

* expires

expires: Thu, 16 May 2019 03:05:59 GMT

在 http 头中设置一个过期时间，在这个过期时间之前，浏览器的请求都不会发出，而是自动从缓存中读取文件，除非缓存被清空，或者强制刷新。缺陷在于，服务器时间和用户端时间可能存在不一致，所以 HTTP/1.1 加入了 cache-control 头来改进这个问题。

* cache-control

cache-control: max-age=31536000

设置过期的时间长度（秒），在这个时间范围内，浏览器请求都会直接读缓存。当 expires和 cache-control 都存在时，cache-control 的优先级更高。

* last-modified / if-modified-since

这是一组请求/相应头

响应头：

last-modified: Wed, 16 May 2018 02:57:16 GMT

请求头：

if-modified-since: Wed, 16 May 2018 05:55:38 GMT

服务器端返回资源时，如果头部带上了 last-modified，那么资源下次请求时就会把值加入到请求头 if-modified-since 中，服务器可以对比这个值，确定资源是否发生变化，如果没有发生变化，则返回 304。

* etag / if-none-match

这也是一组请求/相应头

响应头：

etag: "D5FC8B85A045FF720547BC36FC872550"

请求头：

if-none-match: "D5FC8B85A045FF720547BC36FC872550"

原理类似，服务器端返回资源时，如果头部带上了 etag，那么资源下次请求时就会把值加入到请求头 if-none-match 中，服务器可以对比这个值，确定资源是否发生变化，如果没有发生变化，则返回 304。

#####上面四种缓存的优先级：cache-control > expires > etag > last-modified

####2.使用动态 polyfill
Polyfill 的特点是非必需和不变，因为对于一台手机来说，需要哪些 polyfill 是固定的，当然也可能完全不需要 polyfill。

####3. 使用 SplitChunksPlugin 自动拆分业务基础库

Webpack 4 抛弃了原有的 CommonChunksPlugin，换成了更为先进的 SplitChunksPlugin，用于提取公用代码。

它们的区别就在于，CommonChunksPlugin 会找到多数模块中都共有的东西，并且把它提取出来（common.js），也就意味着如果你加载了common.js，那么里面可能会存在一些当前模块不需要的东西。

而 SplitChunksPlugin 采用了完全不同的 heuristics 方法，它会根据模块之间的依赖关系，自动打包出很多很多（而不是单个）通用模块，可以保证加载进来的代码一定是会被依赖到的。

这就保证了所有公用的模块，都会被抽出成为独立的包，几乎完全避免了多页应用中，重复加载相同模块的问题。


####4. 正确使用 Tree Shaking 减少业务代码体积

例如，我们有下面这样一个使用了 ES Module 标准的模块：

	// math.js
	export function square(x) {
  		return x * x
	}
 
	export function cube(x) {
  		return x * x * x
	}

然后你在另一个模块中引用了它：

	// index.js
	import { cube } from './math'
	cube(123)

经过 webpack 打包之后，math.js 会变成下面这样：

	/* 1 */
	/***/ (function(module, __webpack_exports__, __webpack_require__) {

	"use strict";
	/* unused harmony export square */
	/* harmony export (immutable) */ __webpack_exports__["a"] = cube;

	function square(x) { return x * x; }

	function cube(x) { return x * x * x; }

注意这里 square 函数依然存在，但多了一行 magic comment：unused harmony export square

随后的压缩代码的 uglifyJS 就会识别到这行 magic comment，并且把 square 函数丢弃。

但是一定要注意： webpack 2.0 开始原生支持 ES Module，也就是说不需要 babel 把 ES Module 转换成曾经的 commonjs 模块了，想用上 Tree Shaking，请务必关闭 babel 默认的模块转义：

{
  "presets": [
    ["env", {
      "modules": false
      }
    }]
  ]
}

另外，Webpack 4.0 开始，Tree Shaking 对于那些无副作用的模块也会生效了。

如果你的一个模块在 package.json 中说明了这个模块没有副作用（也就是说执行其中的代码不会对环境有任何影响，例如只是声明了一些函数和常量）：

{
  "name": "your-module",
  "sideEffects": false
}

那么在引入这个模块，却没有使用它时，webpack 会自动把它 Tree Shaking 丢掉：

import yourModule from 'your-module'
// 下面没有用到 yourModule

这一点对于 lodash、underscore 这样的工具库来说尤其重要，开启了这个特性之后，你现在可以无心理负担地这样写了：

import { capitalize } from 'lodash-es';
document.write(capitalize('yo'));

##三、首次内容渲染 -> 可交互（加载及初始化各项组件）

####1. Code Splitting

大多数打包器（比如 webpack、rollup、browserify）的作用就是把你的页面代码打包成一个很大的 “bundle”，所有的代码都会在这个 bundle 中。但是，随着应用的复杂度日益提高，bundle 的体积也会越来越大，加载 bundle 的时间也会变长，这就对加载过程中的用户体验造成了很大的负面影响。

为了避免打出过大的 bundle，我们要做的就是切分代码，也就是 Code Splitting，目前几乎所有的打包器都原生支持这个特性。

Code Splitting 可以帮你“懒加载”代码，以提高用户的加载体验，如果你没办法直接减少应用的体积，那么不妨尝试把应用从单个 bundle 拆分成单个 bundle + 多份动态代码的形式。

比如我们可以把下面这种形式：

	import { add } from './math';
	console.log(add(16, 26));

改写成动态 import 的形式，让首次加载时不去加载 math 模块，从而减少首次加载资源的体积。

	import("./math").then(math => { console.log(math.add(16, 26));});

React Loadable 是一个专门用于动态 import 的 React 高阶组件，你可以把任何组件改写为支持动态 import 的形式。

	import Loadable from 'react-loadable';
	import Loading from './loading-component';

	const LoadableComponent = Loadable({ loader: () => import('./my-component'), loading: Loading,});
	export default class App extends React.Component {
		render() { return <LoadableComponent/>; }
	}

上面的代码在首次加载时，会先展示一个 loading-component，然后动态加载 my-component 的代码，组件代码加载完毕之后，便会替换掉 loading-component。

####2. 编译到 ES2015+ ，提升代码运行效率

##四.可交互 -> 内容加载完毕

####1. LazyLoad

react-lazyload

react-lazy-load

当然你也可以实现像 Medium 的那种加载体验（好像知乎已经是这样了），即先加载一张低像素的模糊图片，然后等真实图片加载完毕之后，再替换掉。

实际上目前几乎所有 lazyload 组件都不外乎以下两种原理：

监听 window 对象或者父级对象的 scroll 事件，触发 load；

使用 Intersection Observer API 来获取元素的可见性。

####2. placeholder

我们在加载文本、图片的时候，经常出现“闪屏”的情况，比如图片或者文字还没有加载完毕，此时页面上对应的位置还是完全空着的，然后加载完毕，内容会突然撑开页面，导致“闪屏”的出现，造成不好的体验。

为了避免这种突然撑开的情况，我们要做的就是提前设置占位元素，也就是 placeholder：

已经有一些现成的第三方组件可以用了：

react-placeholder

react-hold

另外还可以参考 Facebook 的这篇文章：《How the Facebook content placeholder works》

##总结：

这篇文章里，我们一共提到了下面这些优化加载的点：

* 在 HTML 内实现 Loading 态或者骨架屏；
* 去掉外联 css；
* 缓存基础框架；
* 使用动态 polyfill；
* 使用 SplitChunksPlugin 拆分公共代码；
* 正确地使用 Webpack 4.0 的 Tree Shaking；
* 使用动态 import，切分页面代码，减小首屏 JS 体积；
* 编译到 ES2015+，提高代码运行效率，减小体积；
* 使用 lazyload 和 placeholder 提升加载体验。
* ...

实际上可优化的点还远不止这些，这里推荐一些相关资源给大家阅读：

* [GoogleChromeLabs/webpack-libs-optimizations](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2FGoogleChromeLabs%2Fwebpack-libs-optimizations)

* [https://developers.google.com/web/fundamentals/performance/get-started/](https://link.juejin.im/?target=https%3A%2F%2Fdevelopers.google.com%2Fweb%2Ffundamentals%2Fperformance%2Fget-started%2F)

* [A Pinterest Progressive Web App Performance Case Study](https://link.juejin.im/?target=https%3A%2F%2Fmedium.com%2Fdev-channel%2Fa-pinterest-progressive-web-app-performance-case-study-3bd6ed2e6154)

* [A React And Preact Progressive Web App Performance Case Study: Treebo](https://link.juejin.im/?target=https%3A%2F%2Fmedium.com%2Fdev-channel%2Ftreebo-a-react-and-preact-progressive-web-app-performance-case-study-5e4f450d5299)
