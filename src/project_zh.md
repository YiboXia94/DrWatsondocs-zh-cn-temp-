# 项目设置（Project setup）

DrWatson中的部分功能是能够确保在创建和导航项目环境时保持操作的一致性。即使您将项目移动到不同的位置或者不同的计算机上，亦或将其发送给使用不同Julia安装版本的同事，它也能正常工作。另外，在任何使用DrWatson的项目中，导航流程都是相同的。

这就做到了“开箱即用”(TM)，其原因在于遵循了以下原则：

1. **您的科学计算项目同时也是一个由`Project.toml`文件定义的[Julia项目](https://julialang.github.io/Pkg.jl/v1/environments/)** 通过这种方式，项目能够追踪记录使用的包（以及包的版本），并可与任何其他Julia用户共享。
2. **您需要在运行代码前首先激活该项目环境** 这样可以确保您的项目运行在指定的包安装环境中(而不是全局环境)。操作方式请参阅[激活项目](@ref)。
3. **您需要使用DrWatson中的`projectdir`、`datadir`等函数来定位您的项目文件**  请参阅页面[定位项目文件](@ref)。

值得一提的是，我们建议的项目设置方式实现了完全的可复现性，具体请参阅[可复现性](@ref)部分。

## 默认项目设置

DrWatson为任意的科学计算项目提供了一种通用的项目结构设计建议，具体如下:

```@setup project
using DrWatson
struct ShowFile
    file::String
end
function Base.show(io::IO, ::MIME"text/plain", f::ShowFile)
    write(io, read(f.file))
end
```
```@example project
ShowFile(dirname(pathof(DrWatson))*"/defaults/project_structure.txt") # 显示隐藏文件
```

### `src`文件夹 vs `scripts`文件夹
项目中的`src` 和`scripts`文件夹似乎有着相似的功能。然而，这两者之间确实存在着一些区别。我们建议您遵循以下经验法则，来决定将`file.jl`置于其中一个文件夹中：
* 如果在运行 `include("file.jl")` 之后生成了任何内容，无论是数据文件、图像，或是控制台输出，那它就应该被放在`scripts`文件夹中。
* 如果某个功能被多个文件或流程所使用，那它就应该放在`src`中。
* `src` 文件夹应该仅包含定义函数或类型但不输出任何内容的文件。

## 项目初始化

为了按照 [默认项目设置](@ref)部分来进行项目初始化，我们提供了以下函数：

```@docs
initialize_project
```


## 激活项目
DrWatson的这部分功能需要首先激活您的科学计算项目(也就是您的Julia项目)。
这项操作能够有多种实现方式：
   1. 执行代码 `Pkg.activate("path/to/project")`
   2. 使用启动命令 `--project path` 来启动Julia
   3. 设置名称为 [`JULIA_PROJECT`](https://docs.julialang.org/en/latest/manual/environment-variables/#JULIA_PROJECT-1) 的环境变量
   4. 使用DrWatson提供的函数 [`quickactivate`](@ref) 或者宏命令[`@quickactivate`](@ref)
我们推荐您使用第四种方法，尽管它的使用有一些注意事项（详见[`quickactivate`](@ref)部分的文档）

```@docs
quickactivate
@quickactivate
findproject
```

请注意，您可以使用以下方式来获取当前项目的名称：

```@docs
projectname
```


## 在`src`文件夹中包含Julia程序包/模块
请注意， 经由DrWatson初始化的项目并不是一个 Julia程序包，而是一个科学计算项目。换句话说，通常您会希望在项目中开发普通的 Julia程序模块（并且可能后续将它们作为程序包来发布）以便后续可以在使用`using ModuleName`来加载它们。正确的做法是在`src`文件夹中使用包管理器对Julia包进行初始化，具体步骤如下：

1. 激活使用 DrWatson 的项目
2. 切换到项目的根目录（**重要!**）
3. 进入包管理模式并初始化一个自定义程序包：`generate src/ModuleName`
4. 使用包管理器的`dev`功能到`ModuleName` ：`dev src/ModuleName`。请注意这个命令使用了本地路径，具体请参考[this PR](https://github.com/JuliaLang/Pkg.jl/pull/1215)
* 如果您不想将模块作为程序包发布，只需要删除对应的`.git`文件夹即可，文件位置在`src/Modulename/.git`
* 如果您确实计划将这个模块发布为一个Julia程序包，那么就必须将其保存为一个git仓库。此情形下请将`src/ModuleName/.git`置入根目录下的`.gitignore`文件
现在无论何时使用`using ModuleName`，您都将使用本地版本。即使您将项目转移到另一台计算机上，该模组也能正常工作，因为 Manifest.toml 文件中存储了模组所在的本地路径。


## 定位项目文件
为了能够始终一致地定位项目文件位置，DrWatson 提供了核心函数：

```@docs
projectdir
```
除上述函数外，还有以下衍生的函数：

```julia
datadir()
srcdir()
plotsdir()
scriptsdir()
papersdir()
```
这些函数的使用效果与`projectdir`一致，只是将根目录定位到相应的子文件内。由于都是一些被频繁使用的子目录，我们在此提供了相应的函数。

这些函数都使用了`连结路径(joinpath)`，确保了跨不同操作系统创建路径时不会出现错误。我们强烈建议您将子路径作为参数传递给`projectdir`及其衍生函数，而不是在各级路径之间使用乘法符号：
```julia
datadir("foo", "test.jld2") # preferred
datadir() * "/foo/test.jld2" # not recommended
```

### 自定义目录函数
如果您创建了一个需要被频繁访问的目录，那么可以直接自定义一个目录函数方便访问：

```julia
customdir(args...) = projectdir("custom", args...)
```
to make the `customdir` version that works exactly like e.g. `datadir` but for `"custom"` instead of `"data"`.
其工作方式等同于`datadir`，只是它的目录面向`"custom"`而非`"data"`。

## 可复现性
DrWatson建议的项目设置方法能够完美地与Julia标准兼容，使得项目易于共享并且具有完全的可复现性。以下三个原因使其可以实现**真正的**可复现性：
1. `Manifest.toml`的存在使得项目使用的包都被嵌入到项目中。
2. 项目文件夹中的定位均使用了本地目录。
3. 项目本身就是一个Git仓库，这意味着它拥有一个完备的项目历史追溯功能。
如果您将整个项目文件夹发给您的同事，他们只需要进行以下操作：

```julia
julia> cd("path/to/project")
pkg> activate .
pkg> instantiate
```
就可以使用你的项目（*前提是你们使用了同样的Julia安装和版本*）。所有需要的包和依赖项将会被安装，然后您计算机上的脚本将以**相同的方式**在您同事的计算机上运行。

此外，通过DrWatson您可以给每次数值模拟打上“标签”并记录提交ID，详见文档中关于 [`gitdescribe`](@ref)和[`tag!`](@ref)的讨论内容。通过这种方式，任何时候获得的数据结果都可以简单地通过将Git树重置到适当的提交时刻来运行代码复现结果。


## 将现有项目迁移至DrWatson项目
如果您已经拥有一个包含脚本和数据等内容的科学计算项目，则没有必要使用 [`initialize_project`](@ref)函数。
唯一的要求是您项目的所有内容都存储在一个主文件夹中(允许其中包含任意数量的子文件夹)。如果您的项目已经是一个Julia项目（它有自己的Project.toml和Manifest.toml文件），那么您不需要做任何其他操作就可以立即开始使用DrWatson。
尽管我们建议遵循[默认项目设置](@ref)规则，但您也可以选择[自定义目录函数](@ref)。
如果您的项目_不是_一个Julia项目，那么所需的步骤仍然非常简单。您可以执行以下操作：
```julia
julia> cd("path/to/project")
pkg> activate .
pkg> add Package1 Package2 ...
```
当您为项目添加程序包时，Julia会自动为您创建Project.toml和Manifest.toml文件。
