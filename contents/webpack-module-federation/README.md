Webpack Module Federation 实用（避坑）指南  
----  
Webpack Module Federation 功能可以可以让我们把一个应用拆分成多个独立的构建(builds)，每个构建可以独立的开发和部署，不同构建之间可以共享模块。  
本文先用较多篇幅讲诉该功能的原理，然后对相关的配置进行了解释，接着提出了一些注意点，最后是帮助Troubleshooting的内容。

### 原理浅析  
下面将从设计者的角度思考怎么实现远程模块加载和模块共享两个功能点。  

#### 远程模块加载  
怎么实现一个 `remoteImport`  方法，在项目 a 远程加载项目 b 的某个模块呢 ？
```js
// file: project-a/main.js
remoteImport('http://127.0.0.1:80001/project-b/module-1.js').then((module) => {
  console.log(module)
})
```  
我们知道，可以动态创建 `script` 标签来加载远程脚本，例如下面  
```js
// file: remoteImport.js
function loadScript (url) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src=url
    script.onload = resolve
    script.onerror = reject
    document.head.appendChild(script)
  })
}

export default async function remoteImport(url) {
  return loadScript(url)
}
```  
但是这只是加载和执行了脚本，我们没法得到里面给我们导出的对象，怎么办？  

一个简单的方案就是利用全局变量，需要  
* 远程模块要把导出的对象挂到一个自定义的全局变量下面  
* 手动把全局变量名称传给 `remoteImport` 
```diff
// file: project-a/main.js
-remoteImport('http://127.0.0.1:80001/project-b/module-1.js').then((module) => {
+// 加载时传入全局变量名称
+remoteImport('http://127.0.0.1:80001/project-b/module-1.js', 'Module1').then((module) => {
  console.log(module)
})
```

```diff
// file: project-a/remoteImport.js  
function loadScript (url, name) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src=url
-   script.onload = resolve
+   script.onload = () => {
+     // 从全局变量得到导出值
+     resolve(window[name])
+   }
    script.onerror = reject
    document.head.appendChild(script)
  })
}

export default async function remoteImport(url, name) {
  return loadScript(url, name)
}
```  

现在基本实现了远程加载模块，虽然可用，但是不太实用，远程模块路径硬编码，不好控制缓存，虽然可以用协商缓存，但是模块多的话会发送很多请求。

为了更好的控制缓存，我们添加一个层，暂时称之为`远程入口`，它记录了它导出的模块的加载方法和提供一个`get`方法让其他模块来加载它导出的模块，这样我们只需要加载远程入口和调用它提供的`get`方法好了，代码如下    


```js
// file: project-b/remoteEntry.js  
// 模块信息表
const moduleMap = {
  'module-1': () => {
    return loadScript('http://127.0.0.1:8001/project-b/module-1.c093e67e.chunk.js', 'Module1')
  },
  'module-2': () => {
    return loadScript('http://127.0.0.1:8001/project-b/module-2.0e67c0f2.chunk.js', 'Module2')
  }
  //...
}

var ProjectB = {
  get (name) {
    return moduleMap[name]()
  }
}
```
另外，我们增加一个 `entryMap` 对象来记录不同项目的入口地址，并且增加一个加载入口的方法`loadEntry`, 这样我们的 `remoteImport` 传项目名和模块名就好了    
```diff  
// file: project-a/main.js  
-remoteImport('http://127.0.0.1:80001/project-b/module-1.js', 'Module1').then((module) => {
+// 传入项目名和模块名  
+remoteImport('ProjectB', 'module-1').then((module) => {
  console.log(module)
})
```

```diff
// file: project-a/remoteImport.js  
function loadScript (url, name) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src=url
    script.onload = () => {
      resolve(window[name])
    }
    script.onerror = reject
    document.head.appendChild(script)
  })
}

+const entryMap = {
+  'ProjectB': 'http://127.0.0.1:8001/project-b/remoteEntry.js',
+}
+const entries = {}
+async function loadEntry (projectName) {
+  if (!entries[projectName])
+    entries[projectName] = await loadScript(entryMap[projectName], projectName)
+  }
+  return entries[projectName]
+}

export default async function remoteImport(projectName, name) {
-  return loadScript(url, name)
+  // 加载入口
+  await loadEntry(projectName)
+  // 调用入口的 get 方法加载模块
+  return entries[projectName].get(moduleName)
}

```
缓存的问题基本解决了，入口文件的路径是固定的，一般情况下我们不会有非常多的远程入口文件，它的缓存控制可以使用协商缓存策略。  
项目 b 每次构建的时候，生成类似上面的远程入口 `remoteEntry.js`文件就好了。  


#### 模块共享  
上面我们有了一个可以跨项目加载模块的方案，例如可以在 a 项目加载项目 b 的 `module-b` 和 加载项目 c 的 `module-c`, 现在的问题是，如果 `module-b` 和 `module-c` 都使用了 `vue`, 怎么让他们共享一个 `vue`, 而不是加载各自的 `vue` 呢？  


```js
// file: project-b/module-b.js
require('vue')
//....

```

```js
// file: project-c/module-c.js
require('vue')
//....

```

我们需要解决两个问题，
1. 怎么知道某个模块有没有被共享？  
解决：使用一个全局对象来记录被共享的模块信息，入口文件添加一个 `init` 方法，用来把自身共享的模块信息放到该对象上      

2. `require('vue')` 是同步使用，如果这时候再去加载 `vue` 就晚了，怎么确保被使用之前，共享模块已经被加载？ 
解决：共享模块只能被异步模块使用，并且异步模块在被引入的时候，需要添加加载它所依赖的共享模块的代码  

代码如下:  

```diff
// file: project-a/remoteImport.js  
function loadScript (url, name) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src=url
    script.onload = () => {
      resolve(window[name])
    }
    script.onerror = reject
    document.head.appendChild(script)
  })
}

+// 存放共享信息的对象
+window.SharedScope =  window.SharedScope || {}
+function ensureSharedModule(name) {
+  // 共享模块加载
+  return window.SharedScope[name]()
+}

const entryMap = {
  'ProjectB': 'http://127.0.0.1:8001/project-b/remoteEntry.js',
}
const entries = {}
async function loadEntry (projectName) {
  if (!entries[projectName])
    entries[projectName] = await loadScript(entryMap[projectName], projectName)
+   // 初始化
+   entries[projectName].init(window.SharedScope)
  }
  return entries[projectName]
}

export default async function remoteImport(projectName, name) {
  // 加载入口
  await loadEntry(projectName)
  // 调用入口的 get 方法加载模块
  return entries[projectName].get(moduleName)
}

```

```diff
// file: project-b/remoteEntry.js  
// 模块信息表
const moduleMap = {
  'module-b': () => {
+   // 保证 vue 已经被加载
+   return ensureSharedModule('vue').then(() => {
      return loadScript('http://127.0.0.1:8001/project-b/module-b.c093e67e.chunk.js', 'ModuleB')
+   })
  },
  //...
}
+// 共享模块信息
+const sharedModuleMap = {
+ vue: () => {
+   return loadScript('http://127.0.0.1:8001/project-b/vue.6aa6de28.chunk.js', 'Vue')
+ }
+}

var ProjectB = {
  get (name) {
    return moduleMap[name]()
  },
+  init (SharedScoped) {
+    // 把共享的模块信息放到传进来的对象
+    Object.keys(shardModuleMap).forEach(item => {
+      // 如果有值，说明别的项目已经提供了该模块的加载，不需要当前项目的了
+      if (!SharedScope[item]) {
+        SharedScoped[item] = sharedModuleMap[item]
+      }
+    })
+  }
}

```

```js  
// file: project-a/main.js  
// 传入项目名和模块名  
remoteImport('ProjectB', 'module-b').then((module) => {
  console.log(module)
})
```

这样，我们基本实现了模块共享，`remoteImport('ProjectB', 'module-b')` 加载过程大概如下  
* `ProjectB` 入口文件被加载  
* 入口的 `init` 方法被调用，共享模块信息放到了 `SharedScope` 对象  
* 入口的 `get` 方法被调用  
* `moduleMap` 对应 `module-b` 的加载方法执行,  `module-b` 所依赖的共享模块先被加载，然后 `module-b` 才会被加载    

以上就是 webpack module federation 一个简化版的原理。  
在 webpack 的实现中，
* 共享模块给别人用或者使用别人的共享模块的构建(builds)称为`容器(container)`  
* webpack 的 shared scope 里包含了共享模块的版本信息，一个 shared scope 里可能包含多个版本的共享模块，以满足不同项目的版本需求  
* 通过配置可以有多个 shared scope，默认只有一个，名称叫 `default`, 不同的 shared scope 可以实现类似 a 和 b 项目共享一个 `vue@3.x`, c 和 d 项目共享一个 `vue@2.x` 的功能  
* 远程入口是 webpack 根据你的配置构建出来的  
* 某一个 shared scope 在用到的时候才被初始化；远程入口在某个 shared scope 初始化之前被加载    
* 某个模块如果用到了共享模块，webpack 会保证那个模块所依赖的共享模块先被加载，例如  
假设下面的 `App.js` 依赖了共享的 `react`  
```js
import('./App.js')
```
构建之后*可能*会变成类似下面这样  
```js  
Promise.all(/*! import() */[
  __webpack_require__.e("webpack_sharing_consume_default_react_react"),
  __webpack_require__.e("src_App_js")
])
.then(__webpack_require__.bind(__webpack_require__, /*! ./App.js */ "./src/App.js"));
```
`webpack_sharing_consume_default_react_react` 这个名称由下面提到的一些配置构成，`webpack_sharing_consume_{sharedScope}_{shareKey}_{包名}`。在加载某个共享模块的时候，会初始化它所属的 shared scope  


1. 如果不同的模块需要不同的共享库的版本怎么办，例如，`module-b` 需要 `vue@2.x`, 但是 `module-c` 需要  `vue@3.x`？  
我们可以通过在全局共享对象中添加版本信息    
2. 如果我们想要项目 a, 项目 b 共享 `vue`, 项目 c, 项目 d 共享另外一个  `vue`？  
我们可以创建多个共享对象表，共享模块可以配置它所在的共享对象表的名称  


### 配置说明  
下面列出 webpak module federation 的一些配置项以及解释。  

```js
// ...
module.exports = {
  // ...
  plugins: [
    new ModuleFederationPlugin({
      name: "app1", // 当前容器的名称，加载的时候会创建跟这个名称一样的全局变量，注意不要冲突    
      filename: 'remoteEntry.js', // 远程入口的名称  
      remotes: {
        // 远程容器的名称和入口地址， 
        // 需要注意的是，
        // 前面这个 'app2' 可以随意定义，用于 import，例如 import('app2/some-module')  
        // 后面这个 'app2' 必须是这个容器的名称，
        // 就像我们上面说的一样，webpack 会用容器名称创建一个全局变量，加载完入口之后，需要用这个变量来得到容器的信息
        // 这里也可以用 promise 动态 url ， 详情看文档 https://webpack.js.org/concepts/module-federation/#promise-based-dynamic-remotes
        app2: "app2@http://localhost:3002/remoteEntry.js",  
      },
      exposes: {
        // 暴露的模块名称以及路径，例如这个模块可以在别的项目通过 import('app1/App') 引入
        './App': './src/App.js' 
      },
      shared: {
        react: { // 共享的模块名，在 import 共享模块时需要用这个名称，webpack 默认会用这个 key 找对应的包，如果这里配的不是包名，需要在下面的 import 属性指明真正的包名  
          eager: true, // 配置之后，模块会在 index.html 里面的直接加载，而不是通过异步加载  
          // import: 'react', // 被导入的模块名称，默认值是当前对象的属性名，例如这里的 react
          shareKey: 'react', // 在 shared scope 对象里的 key
          shareScope: 'myScope' // 默认 'default', 当前模块所属的 shared scope 名
          singleton: true, // 设为 true 可使该模块在它的 shared scope 里只有一个版本
          version: '17.0.2', // 当前提供的模块的版本号
          requiredVersion: '^17.0.2', // 需要的版本, 默认通过 package.json 里得到
          // packageName: 'react',  // 用于在 package.json 中查明当前模块需要的版本
          strictVersion: true, // 设为 true 之后，只有当 shared scope 里面别的 container 已加载的该模块的版本严格满足当前配置的 requredVersion ，当前 container 才会使用
        },
      },
    }),
  ]
}
```

### 需要注意的点    
* 不同容器的 package.json 包名不能相同  
  相同可能会因为冲突而导致奇怪的错误    
* 容器名称要唯一，避免与其他全局变量冲突  
* 远程的 container 的 `output.publicPath` 建议不要配置或者配成 `auto`  
  配置不恰当的 publicPath 会导致加载失败，不配置的话，webpack 会自动根据远程入口的 src 地址来决定。  
  也可以通过 `__webpack_public_path__` 变量动态设置，详情查看 [Dynamic Public Path](https://webpack.js.org/concepts/module-federation/#dynamic-public-path)  
* 远程的 `optimization` 配置
  * `optimization.runtimeChunk` 建议不要设  
  * `optimization.splitChunks.splitChunks` 建议不要设或设成 'async'     
  * `optimization.splitChunks.maxSize` 建议不要设   
  设置不恰当可能会导致入口文件缺少一些运行时代码，从而加载失败  
* eager 的模块里面同步import的共享模块也必须 eager  
  因为同步import的模块需要比当前模块早加载完
* 非 eager 的共享模块只能异步import或者在异步import模块中使用  
  非 eager 模块会放在 async chunk 中
* 目前还不支持热模块替换  
  截止目前（2021年9月3日），HMR for Module Federation 还没实现，可以在 [这里](https://webpack.js.org/vote/) 投票以提高实现这个功能的优先级。


### Troubleshooting  
#### Uncaught Error: Shared module is not available for eager consumption  
共享模块只能在异步模块中使用，eager 模块所依赖的模块也必须 eager。检查是不是有非异步模块  
* eager 的模块里面同步import的共享模块也必须 eager  
* 非 eager 的共享模块只能异步import或者在异步import模块中使用  

#### Uncaught (in promise) ScriptExternalLoadError: Loading script failed   
* 不同的 container 的 package.json 的包名有重复？
* exposes 里面的容器全局变量是否拼写错误？  
* container 的名称是否有重复或者跟别的全局变量冲突？
* optimzation 配置是否不恰当？  

其他请看官方  [Troubleshooting](https://webpack.js.org/concepts/module-federation/#troubleshooting)


### 其他  
更多的场景和使用方式的可以参考官方的示例 https://github.com/module-federation/module-federation-examples


--  

参考  
[Webpack Module Federation Documentation](https://webpack.js.org/concepts/module-federation/)  
[ModuleFederationPlugin Documentation](https://webpack.js.org/plugins/module-federation-plugin/)  
