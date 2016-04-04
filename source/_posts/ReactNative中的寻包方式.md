title: ReactNative打包器中的寻包方式
date: 2016-03-31 18:52:12
tags:
	- ReactNative
---

> ReactNative中经常能见到不同于Node.js中原生的包引用方式的引用语句。平时我们要么是通过包名，要么是通过js文件的绝对路径来进行引用。而在RN中，有些组建所在的目录非常深，但仅凭一个组件名就可以准确的引用到组件，那么打包器是如何完成这项工作的呢


<!-- more -->

---

# 问题起因

几天前，我在用ReactNative写APP的过程中，用到ListView，并且要求ListView中的子控件是自行去获取数据并且渲染的，但当我把在外面表现好好的Component放到ListView的行中的时候，却发现不能获取数据，在查看了ListView的js侧的实现代码后，发现官方的ListView其实是渲染成了原生的ScrollView，并且在渲染粘性头部和行中的内容时，为了复用和性能，都包裹了一层StaticRender（关于StaticRender的内容，可以看本文结尾）。这层StaticRender给我造成了很大的障碍，所以我决定自己去掉这层外壳。

最开始是直接在官方的代码上做的修改，后来觉得还是自己复制出来再修改为好，以免给之后的维护带来麻烦。但就在这时麻烦出现了。当我尝试使用我自定义的ListView的时候，Packager直接给报错

    Error building DepdendencyGraph:
    Error: Naming collision detected
    
# 命名冲突

很明显看出来，Packager在构建依赖的时候提示说命名冲突。在我引用我的自定义Component时，我是直接使用相对路径进行的引用

    import ListView from '/component/customListView.js';
    
那我们再来看看引用原生的ListView时，我们一般是怎么做的

    import {ListView} from 'react-native';
    
为什么这样完全截然不同的引用方式能在没有同时引用的文件中报错呢，换句话说，打包器是如何检测的呢

# 打包器的寻包方式

我们先来看一个我们平时司空见惯的写法

    import ActivityIndicatorIOS from 'ActivityIndicatorIOS';

我们都知道，不管是在ES6的原生模块支持也好，还是像Webpack那样，我们一般都是直接引用js文件的相对路径或者使用node包名。那么RN的打包器是如何找到对应的包的呢。

> The packager uses two methods to look up modules. The first is based on docblock headers: if you write "@providesModule X" in the first docblock this enables require('X'). The other method is Node's resolution.

这是在Github上的一个Issue上看到的回答，打包器使用两种方式来寻包，第一种就是Node.js的原生方式，另一个是在js文件中头部的DocBlock区中声明如下的两行中的一行（这两行的效果是完全相同的）

    @provides {定义的包名}
    @providesModule {定义的包名}
    
这个句子要包裹在多行注释中，/*  */的形式，单行注释中包含这句话是无效的。

# 打包器的实现

按理说说到这里就可以了，但都到这了，不看看打包器自身是如何实现的，是说不过去的，我们来看看他的代码

这是打包器在构建依赖是的代码，可以看到在读完文件之后，会解析文件内容，，如果包含我们上述的那两句，就会以这个包名来进行依赖分析。

```javascript
readFile(modulePath, 'utf8')
  .then(function(content) {
    var moduleDocBlock = docblock.parseAsObject(content);
    if (moduleDocBlock.providesModule || moduleDocBlock.provides) {
      moduleData.id = /^(\S*)/.exec(
        moduleDocBlock.providesModule || moduleDocBlock.provides
      )[1];

      // Incase someone wants to require this module via
      // packageName/path/to/module
      moduleData.altId = self._lookupName(modulePath);
    } else {
      moduleData.id = self._lookupName(modulePath);
    }
    moduleData.dependencies = extractRequires(content);

    module = new ModuleDescriptor(moduleData);
    self._updateGraphWithModule(module);
    return module;
  });
};
```

# StaticRender

首先声明这一节与本次讨论的内容没什么关系，只是解释一下这个概念，其实StaticRender的实现十分简单，我们直接上包的代码：

```javascript
var StaticRenderer = React.createClass({
  propTypes: {
    shouldUpdate: React.PropTypes.bool.isRequired,
    render: React.PropTypes.func.isRequired,
  },

  shouldComponentUpdate: function(nextProps: { shouldUpdate: boolean }): boolean {
    return nextProps.shouldUpdate;
  },

  render: function(): ReactElement {
    return this.props.render();
  },
});
```

可以看到，这个Component主要是通过shouldComponentUpdate来实现的组建静态，然组建的渲染效果仅仅取决于props，成为props的纯函数。

再者，ListView对外的包装中，并没有对StaticRender中的shouldUpdate对外进行了预留，所以在ListView中我们的所有Component的都是props所决定的一次性的结果，不能进行re-render。