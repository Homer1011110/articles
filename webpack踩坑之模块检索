# 问题描述

项目中使用了一个npm包a。前几天一直用得好好的，突然某次拉了别的分支代码后，就出Bug了。

第一反应是别人把这个包的版本变了。查看了下项目的`package.json`、`package-lock.json`文件，该模块和依赖模块的信息并没有改变，`node_modules/a`中的版本信息也和`package.json`中的对应。

一下子没了头绪，只好到node_modules中去调试一下。

# TL;DR;

拉到最后看总结 XD

# node_modules目录结构

项目中`node_modules`目录如下:

```
node_modules
│
└───a
│   │   index.js
|   |   ...
│   │
│   └───node_modules
│       │   ...
│       └───c
|           |   index.js
|           |   ...
│   
└───c
    │   index.js
    │   ...
```

从该目录结构中可以发现，模块a的目录下还有一个`node_modules`目录，这个目录里放的是模块a的依赖。眼尖的同学可能发现了，`项目本身的node_modules`目录和`a模块的node_modules`目录中都有安装了模块c。这是为什么呢？

原因有2个:

1. 项目直接依赖了模块c
2. 项目没有直接依赖模块c，但是项目直接依赖的模块b中依赖了模块c，并且和a中依赖的模块c版本不兼容。

我们的项目中并没有直接引用模块c，所以是第二种情况。

## npm的模块安装机制

本节主要解释为什么项目没有直接依赖模块c，却会把c安装在项目的node_modules目录下。不感兴趣的同学可以直接跳过。

假设项目依赖了模块a和模块b，模块a依赖模块c的1.0.0版本，模块b依赖模块c的2.0.0版本。

### npm2

在npm2的时候，使用嵌套的方式来安装模块，c模块分别被安装到a和b模块的node_modules目录中。

![npm2](https://assets.processon.com/chart_image/5d0f5167e4b0205a6f4c8a80.png)

这种方式虽然简单，但是却会导致node_modules中存在大量相同的模块。想象一下，如果模块a和模块依赖的模块c都是1.0.0版本，使用这种方式就会产生冗余的模块。

### npm3

到npm3的时候，npm2中产生冗余模块的情况得到改善。npm安装模块时会尽量把模块安装到最外层的node_modules目录中，让模块能够尽量被复用。

1. 安装模块时，如果外层node_modules目录中没有同名模块，就将其安装到最外层ndoe_modules目录中
2. 如果外层node_modules目录中已经存在了同名模块，并且**版本兼容**，则不再安装（使用时直接使用外层模块）
3. 如果外层node_modules目录中已经存在了同名模块，并且**版本不兼容**，则安装在父模块的node_modules目录中

上述情况的安装模块如图

![npm3](https://assets.processon.com/chart_image/5d0f56eae4b0d4ba3540eded.png)

# 引用了错误的模块

到`node_modules/a/node_modules/c/index.js`中加了一些log，发现居然没执行！？

到这一步，要么是log的位置没写对，要么是没有引入这个模块。确认了webpack配置中的`resolve.mainFields`属性和模块c的`package.json`文件信息后，排除了第一种可能。

Tips: `resolve.mainFields`属性用来告诉webpack，引入一个npm模块时，如何找到这个模块的入口。

这时候已经有点懵了，引用模块时不是先从当前目录下的node_modules目录中开始一级一级向上查到吗？说到向上查找，那便到上一级的node_modules目录中去试一试。

果然！引用的是最外层node_modules中的模块c！

难道webpack查找模块的方式和Node.js不一样吗？还是因为webpack的某些配置导致的？

使用如下webpack配置来构建，发现并没有存在上述问题。

```javascript
const path = require('path')
module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'main.js',
    path: path.resolve(__dirname, './dist/js')
  },
};
```

那接下来只要排查到底是哪些webpack配置影响到模块检索。查看项目中的webpack配置，和模块检索相关的只有`resolve`属性。

```javascript
const config = {
  resolve: {
    modules: [
        path.resolve(projectDir, 'src'),
        path.resolve(projectDir, 'node_modules'),
        path.resolve(imtPath, 'node_modules'),
    ],
    // es tree-shaking
    mainFields: ['jsnext:main', 'browser', 'main'],
    alias: {},
    extensions: ['.jsx', '.js'],
  }
}
```

所幸配置不多，对着webpack文档查一下，很快便找到了问题：`resolve.modules`中使用了绝对路径。以下为webpack文档原文：

>告诉 webpack 解析模块时应该搜索的目录。
>
>绝对路径和相对路径都能使用，但是要知道它们之间有一点差异。
>
>通过查看当前目录以及祖先路径（即 ./node_modules, ../node_modules 等等），相对路径将类似于 Node 查找 'node_modules' 的方式进行查找。
>
>使用绝对路径，将只在给定目录中搜索。

上述webpack配置中，`path.resolve(projectDir, 'node_modules')`为项目的node_modules目录。这样配置的原因，是因为想要优化模块检索的速度，结果却导致了这么严重的Bug。

根据webpack文档，就是因为这个绝对路径导致了Bug。那么只要把这个绝对路径换成`node_modules`，Bug便解决了。

# 总结

在使用webpack构建js时，有的同学可能会为了加快构建速度，使用`resolve.modules`来指定项目的node_modules目录。

npm在安装模块时，会优先将包安装在项目node_modules目录的最外层，除非有冲突才会安装到父npm模块下的node_modules目录中。

而webpack配置中的`resolve.modules`设置成项目node_modules目录的绝对路径时，会导致webpack在查找node_modules目录时，只在最外层目录查找，忽略掉更深层次的同名模块。这与默认的查找策略“**优先使用深层模块**”相反，导致构建时使用了错误的npm包。

