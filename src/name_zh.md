# 模拟命名规则
我们在此简要介绍函数的命名功能, 从而能够帮助你快速创建模拟的参数容器，并以一种直观且一致的方案来命名你的模拟。  
## 命名方案
我们提供了一种强大的命名方案，使得您能够快速地进行模拟项目命名、新建模拟列表、以及检查已完成的模拟工作等等。更重要的是，它能够让您轻松地创建基于模拟本身的项目名称，且该名称具有**一致性**和**决定性**。

这就是函数[`savename`](@ref)提供的功能。当然，你不光可以用它来给存储的数据命名，你也可以用它来实现任何你想实现的功能，例如编制数据表格的时候给数据列添加标签。

此外，[`savename`](@ref)同样适用于给图片创建对应的标题。例如：`savename(c; connector = ", ")`。

```@docs
savename
```

请注意，这个命名方案完美集成了Parameters.jl的功能。

## 便利函数
我们提供了一些便利函数，以便使用者减少全局函数的调用并且能够方便地创建元组、字典，同时做到在两者之间的快速切换。

```@docs
@dict
@strdict
@ntuple
@savename
ntuple2dict
dict2ntuple
tostringdict
tosymboldict
```

请注意，我们同样在程序包里提供了[UnPack.jl](https://github.com/mauro3/UnPack.jl)中便捷的`@pack!, @unpack`工具，原因是这两个宏对于 [`@dict`](@ref)类型以及类似函数的兼容性非常好。请知悉两者在使用逗号`,`的语法区别：`d = @dict a b c` 和`@unpack a, b, c = d` 。

```@docs
@unpack
@pack!
```

## 对 `savename`进行个人定制
您可以自行定制[`savename`](@ref)。例如，您可以让它只使用几个特定的关键词而非全部，抑或是让它以不同的方式来访问数据（甚至加载文件！）。你甚至可以让它拥有自定义的`前缀`！

为此，您可以扩展以下几个函数：
```@docs
DrWatson.allaccess
DrWatson.access
DrWatson.allignore
DrWatson.default_allowed
DrWatson.default_prefix
DrWatson.default_expand
```

关于定制`savename`的例子请参考[现实案例](@ref)。
值得一提的是，我们推荐您阅读 [`savename` and nested containers](@ref)部分来学习如何对`savename`进行逆向工程。

## 对`savename`进行逆向工程
```@docs
parse_savename
```
