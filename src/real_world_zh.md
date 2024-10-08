# 现实案例

## 易于使用的本地目录

当您使用[`initialize_project`](@ref)函数按照DrWatson推荐的方式设置您的科学计算项目，那么每个项目中的每个文件都会以如下方式开头：

```julia
using DrWatson
@quickactivate "MagneticBilliardsLyapunovs"
using DynamicalBilliards, GLMakie, LinearAlgebra

include(srcdir("plot_perturbationgrowth.jl"))
include(srcdir("unitcells.jl"))
```
在所有项目中，均使用`datadir/plotsdir`保存数据和绘图：
```julia
@tagsave(datadir("mushrooms", "Λ_N=$N.jld2"), (@strdict(Λ, Λσ, ws, hs, description)))
```
这种方法的优点是，无论您将特定文件移动到不同的子文件夹中（这种情况经常发生），还是将整个项目文件夹移动到其他地方，它都可以正常工作！
**请确保您已经理解使用[`quickactivate`](@ref)的注意事项！**
这是另一个项目的示例。您会注意到它另一个优点：即使存在不同的项目，您也可以使用相同的语法访问数据或源文件夹！
```julia
using DrWatson
@quickactivate "EmbeddingResearch"
using Parameters
using TimeseriesPrediction, LinearAlgebra, Statistics

include(srcdir("systems", "barkley.jl"))
include(srcdir("nrmse.jl"))

# stuff...

save(datadir("sim", "barkley", "astonishing_results.jld2"), data)
```

## 将您的项目转化为一个可用的模组
对于某些项目，通常情况下需要在 _项目中的每个文件_ 的开头加载源文件夹中的一些包和文件。举例来说，我项目中  _任何_ 编写的脚本的前五行代码都会是：

```julia
using DrWatson
@quickactivate "AlbedoProperties"
using Dates, Statistics, NCDatasets
include(srcdir("core.jl"))
include(srcdir("style.jl"))
```
将所有命令放在一个代码文件中并加载该文件会对我们来说十分方便，例如执行`include(srcdir("everything.jl"))`来运行包含所有指令的文件。
但是，我们可以做得更好！得益于Julia处理项目和模块路径的方式，我们事实上可以当前活动的项目转换为可用的模组。如果在`src`文件夹中定义了一个`AlbedoProperties.jl`文件，其中包含一个名为`AlbedoProperties`的模组（请注意模组名称必须与项目文件名称 _完全匹配_）,那么当执行 `using AlbedoProperties`时，Julia实际上只是将这个模组引入作用域。
因此，对于一些需要类似操作的项目，我最终的操作是创建上述的文件并在其中添加以下内容：
```julia
module AlbedoProperties

using Reexport
@reexport using Dates, Statistics
using NCDatasets: NCDataset, dimnames, NCDatasets
export NCDataset, dimnames
include("core.jl") # this file now also has export statements
include("style.jl")

end
```
然后，所有代码文件的开头都转换为：
```julia
using DrWatson
@quickactivate :AlbedoProperties
```
它利用了 [`@quickactivate`](@ref)的特性，实质上结合了`@quickactivate "AlbedoProperties"`和`using AlbedoProperties`两个命令的功能。

## `savename` and 项目标签
The combination of using [`savename`](@ref) and [`tagsave`](@ref) makes it easy and fast to save output in a way that is consistent, robust and reproducible. Here is an example from a project:

结合使用[`savename`](@ref)和[`tagsave`](@ref)可以轻松快速地保存输出结果，并且保证了结果的一致性、稳健性和可重复性。以下是一个项目的示例：
```julia
using DrWatson
quickactivate(@__DIR__, "EmbeddingResearch")
using TimeseriesPrediction, LinearAlgebra, Statistics
include(srcdir("systems", "barkley.jl"))

ΔTs = [1.0, 0.5, 0.1] # resolution of the saved data
Ns = [50, 150] # spatial extent
for N ∈ Ns, ΔT ∈ ΔTs
    T = 10050 # we can offset up to 1000 units
    every = round(Int, ΔT/barkley_Δt)
    seed = 1111

    simulation = @ntuple T N ΔT seed
    U, V = barkley(T, N, every; seed = seed)

    @tagsave(
        datadir("sim", "bk", savename(simulation, "jld2")),
        @strdict U V simulation
    )
end
```
保存的文件长这样：
```
path/to/project/data/sim/bk_N=50_T=10050_seed=1111_ΔT=1.jld2
```
每个文件都是字典类型，包含了`:U, :V, :simulation`数据字段。同时，文件还包含了`:gitcommit, :script`等字段。当我读取这个文件时，我就能准确地知道是什么代码生成了它（前提是我没有忘记定期提交代码更改:P）。

## [自定义 `savename`](@id customizing_savename)
以下是自定义[`savename`](@ref)的简单示例。我们在此使用了一个通用的数据结构`Experiment`来记录猫鼠实验。
首先我们定义相关的数据类型：
```@example customizing
using DrWatson, Dates
using Base: @kwdef # for defining structs with keyword values

# Define a type hierarchy we use at experiments
abstract type Species end
struct Mouse <: Species end
struct Cat <: Species end

# @with_kw comes from Parameters.jl
@kwdef struct Experiment{S<:Species}
    n::Int = 50
    c::Float64 = 10.0
    x::Float64 = 0.2
    date::Date = Date(Dates.now())
    species::S = Mouse()
    scientist::String = "George"
end

e1 = Experiment()
e2 = Experiment(species = Cat())
```
为了分析我们的实验，我们需要实验使用的物种信息，并且为了在稍后使用多重分派功能，我们决定将这些信息与类型相关联。这就是我们定义 `Species`的原因。
现在,我们想要自定义[`savename`](@ref)。我们首先扩展[`DrWatson.default_prefix`](@ref)：
```@example customizing
DrWatson.default_prefix(e::Experiment) = "Experiment_"*string(e.date)

savename(e1)
```
但是，这还不够好。因为[`savename`](@ref)中没有包含物种信息，且日期信息也重复了。我们必须扩展[`DrWatson.default_allowed`](@ref)来指定在`savename`中应该扩展的数据类型：
```@example customizing
DrWatson.default_allowed(::Experiment) = (Real, String, Species)

savename(e1)
```
为了更好地输出`Species`信息，我们可以扩展`Base.string`，这是DrWatson在 [`savename`](@ref)中内置的显示数值的方法。
```@example customizing
Base.string(::Mouse) = "mouse"
Base.string(::Cat) = "cat"
savename(e1)
```
最后，假设实验中哪位科学家执行的信息对于`savename`来说并不重要。我们可以扩展最后一个方法[`DrWatson.allaccess`](@ref)：
```@example customizing
DrWatson.allaccess(::Experiment) = (:n, :c, :x, :species)
```
这样只有这四个字段将被使用(注意`date`字段已经在`default_prefix`中被使用了)。最终我们可以得到:
```@example customizing
println( savename(e1) )
println( savename(e2) )
```

## `savename` 和嵌套容器
对于用户定义的结构体和相当复杂的项目，通常需要将其他容器作为子字段包含在“主”容器中。`savename`也可以适应这些情况。
考虑以下示例，其中我需要一个核心结构来表示一个时空系统模型及其数值模拟：
```@example customizing
struct SpatioTemporalSystem
    model::String # system codeword
    N        # Integer or Tuple of integers: spatial extent
    Δt::Real # sampling time in real time units
    p        # parameters. nothing or Dict{Symbol}
end
const STS = SpatioTemporalSystem

struct SpatioTemporalTimeseries
    sts::STS
    T::Int       # total frame amount
    ic           # initial condition (matrix, string, seed)
    fields::Dict # resulting timeseries, dictionary of string to vector
end
const STT = SpatioTemporalTimeseries
```
在我的用例中，`p` 可以是`nothing`，也可以是包含了时空模型的参数集的字典。
为了让`savename`适用于这种情况，我们使用[`DrWatson.default_expand`](@ref)的相关功能。
扩展所需的方法使得我可以这样做：
```@example customizing
DrWatson.allaccess(c::STS) = (:N, :Δt, :p)
DrWatson.default_prefix(c::STS) = c.model
DrWatson.default_allowed(c::STS) = (Real, Tuple, Dict, String)
DrWatson.default_expand(c::STS) = ["p"]

bk = STS("barkley", 60, 0.1, nothing)
savename(bk)
```
当我想要使用与默认值不同的参数时
```@example customizing
a = 0.3; b = 0.5
bk = STS("barkley", 60, 0.1, @dict a b)
savename(bk)
```
扩展到第二个结构体也没问题：
```@example customizing
DrWatson.default_prefix(c::STT) = savename(c.sts)
stt = STT(bk, 1000, nothing, Dict("U"=>rand(100), "V"=>rand(100)))
savename(stt)
```

## 不再有“我到底有没有跑过这个模拟”的焦虑
有那么一段代码让你感觉烦躁：你可能跑过也可能没有，可能保存了生成的数据也可能没有，那么你会不断自问：“我到底有没有跑过这个模拟？”。根据运行代码的成本高低，拥有一个良好的框架来回答这个问题可能非常重要！
这就是 [`produce_or_load`](@ref)的作用。您可以将代码包装在一个函数中，然后 [`produce_or_load`](@ref) 将为您处理剩下的工作！我发现这项功能在生成论文数据图时特别有用。
以下是一个例子；原来我有这样一段代码：
```julia
HTEST = 0.1:0.1:2.0
WS = [0.5, 1.0, 1.5]
N = 10000; T = 10000.0

toypar_h = [[] for l in WS]
for (wi, w) in enumerate(WS)
    println("w = $w")
    for h in HTEST
        toyp = toyparameters(h, w, N, T)
        push!(toypar_h[wi], toyp)
    end
end
```
运行这段代码需要花费几分钟的时间。为了使用[`produce_or_load`](@ref)函数，我必须首先将此代码包装在高级函数中，如下所示：
```julia
function simulation(config)
    HTEST = 0.1:0.1:2.0
    WS = [0.5, 1.0, 1.5]
    @unpack N, T = config
    toypar_h = [[] for _ in WS]

    for (wi, w) in enumerate(WS)
        println("w = $w")
        for h in HTEST
            toyp = toyparameters(h, w, N, T)
            push!(toypar_h[wi], toyp)
        end
    end
    return @strdict toypar_h
end

N = 2000; T = 2000.0
data, file = produce_or_load(
    datadir("mushrooms", "toy"), # path
    @dict(N, T), # container
    simulation; # function
    prefix = "fig5_toyparams" # prefix for savename
)
@unpack toypar_h = data
```
现在，每当我运行此代码块时，函数都会自动测试文件是否存在。仅当它不存在时，代码才会，同时新的结果将被保存以确保我不必再次运行它。

额外的步骤是我必须从容器`file`中提取我需要的有用数据。幸好有[`@unpack`](@ref)宏，或者如果您使用Julia v1.5或更高的版本，可以使用命名解构语法，`(; a, b) = config`，使得解包变得非常容易。

## 用哈希码来`produce_or_load`

如上所示， 默认情况下[`produce_or_load`](@ref) 使用的是 [`savename`](@ref)从配置输入中提取文件名。此文件名用于检查程序是否已运行并保存了其输出。但是，在某些情况下，您可能有太多参数或复杂的嵌套结构，仅使用[`savename`](@ref)不能对它们进行编码或者并不是那么方便。
幸运的是，我们可以使用基础Julia的`hash`函数代替[`savename`](@ref)，我们将在以下示例中进行说明。

```@example customizing
using DrWatson
using Random

function sim_large_c(config)
    @unpack x, f = config
    r = sum(x)*f.a + f.t.b + f.t.c
    return @strdict(r)
end

## Some nested structs
f1 = (a = 1, t = (b = 2, c = 3))
f2 = (a = 2, t = (b = 4, c = 5))
## some containers with too many parameters
rng = Random.MersenneTwister(1234)
x1 = rand(Random.MersenneTwister(1234), 1000)
x2 = randn(Random.MersenneTwister(1234), 20)

preconfigs = Dict("x" => [x1, x2], "f" => [f1, f2])
configs = dict_list(preconfigs)

path = mktempdir()
pol_kwargs = (prefix = "sim_large_c", verbose = false, tag = false)

for config in configs
    produce_or_load(sim_large_c, config, path; pol_kwargs...)
end

readdir(path)
```
如您所见，这显然毫无用处（囧） 。`savename`没有从给定的`config`容器中返回任何内容，因此所有数据都具有相同的名称。
让我们改用`hash`:
```@example customizing
rm(joinpath(path, "sim_large_c.jld2"))
for config in configs
    produce_or_load(sim_large_c, config, path; filename = hash, pol_kwargs...)
end
readdir(path)
```
很好。但是，为了谨慎起见，如果我们使用具有相同类型和大小的不同输入`x`，是否会得到不同的文件名（如期望的那样）？
```@example customizing
config = Dict("x" => rand(Random.MersenneTwister(4321)), "f" => f1)
produce_or_load(sim_large_c, config, path; filename = hash, pol_kwargs...)
readdir(path)
```
是的。但是，如果我们使用完全相同的数字和函数，是否会产生完全相同的哈希码 ？，因此也不会重新运行模拟（如期望的那样）？
```@example customizing
config = Dict("x" => rand(Random.MersenneTwister(1234), 1000), "f" => f1)
produce_or_load(sim_large_c, config, path; filename = hash, pol_kwargs...)
readdir(path)
```
完美！

!!! warning "谨慎使用`hash`。"
	`hash`函数的局限性也适用于此情形。例如,自定义类型应实现`==`以确保`hash`能按预期工作。
	通常，应避免在函数中使用`hash`。函数的哈希基于函数的名称，因此它不会捕获关于函数实际代码或其方法的信息。因此，只有在函数是来自Base Julia等已建立的名称（如sin、cos等）时，才能使用此方法。你也 _完全_ 不能使用匿名函数，因为即使在同一Julia会话中以相同方式定义它们，相应的`hash`也不同。

## 模拟前的准备 & 运行模拟

### 准备字典变量

这是一个使用 [`dict_list`](@ref) 的项目的简短脚本示例：
```@example customizing
using DrWatson

general_args = Dict(
    "model" => ["barkley", "kuramoto"],
    "noise" => 0.075,
    "noisy_training" => [true, false],
    "N" => [100],
    "embedding" => [ #(γ, τ, r, c)
    (4, 5, 1, 0.34), (4, 6, 1, 0.28)]
)
```

```@example customizing
dicts = dict_list(general_args)
println("Total dictionaries made: ", length(dicts))
dicts[1]
```
Also, using the type [`Derived`](@ref), we can have parameters that are computed depending on the value of other parameters:
此外，使用 [`Derived`](@ref) 类型，我们可以得到依赖于其他参数值的计算参数：
```@example customizing
using DrWatson

general_args2 = Dict(
    "model" => "barkley",
    "noise" => [0.075, 0.050, 0.025],
    "noise2" => [1.0, Derived(["noise", "N"], (x,y) -> 2x + y)],
    "noisy_training" => true,
    "N" => 100,
)
```
```@example customizing
dicts2 = dict_list(general_args2)
println("Total dictionaries made: ", length(dicts2))
dicts2[1]
```
现在，您可以自己决定如何使用这些字典。通常，每个字典都会被传递给一个像 `main` 这样的 Julia 函数，该函数会提取必要的数据并调用必要的函数。

假设我编写了一个函数，它接受其中一个字典并将文件保存在本地某个位置：
```@example customizing
function cross_estimation(data)
    γ, τ, r, c = data["embedding"]
    N = data["N"]
    # add fake results:
    data["x"] = rand()
    data["error"] = rand(10)
    # Save data:
    prefix = datadir("results", data["model"])
    get(data, "noisy_training", false) && (prefix *= "_noisy")
    get(data, "symmetric_training", false) && (prefix *= "_symmetric")
    sname = savename((@dict γ τ r c N), "jld2")
    mkpath(datadir("results", data["model"]))
    save(datadir("results", data["model"], sname), data)
    return true
end
```
### 使用 map 和 pmap 函数
运行多个模拟的一种方式是使用`map`(使用`pmap`的过程相同)。要运行所有模拟，只需执行以下操作：
```@example customizing
dicts = dict_list(general_args)
map(cross_estimation, dicts) # or pmap

# load one of the files to be sure everything is ok:
filename = readdir(datadir("results", "barkley"))[1]
file = load(datadir("results", "barkley", filename))
```

### 使用串行集群
如果我不能将 `dict_list` 的结果存储在内存中，那么我必须更改方法，从磁盘加载它们。这可以通过 [`tmpsave`](@ref) 函数轻松实现。

与使用Julia通过`map/pmap`在一个进程中运行所有工作不同，可以使用Julia将多个工作提交到集群队列中。对于上面的示例，执行此操作的Julia程序如下所示：

```julia
dicts = dict_list(general_args)
res = tmpsave(dicts)
for r in res
    submit = `qsub -q queuename julia runjob.jl $r`
    run(submit)
end
```
现在文件`runjob.jl`  的内容如下：
```julia
f = ARGS[1]
dict = load(projectdir("_research", "tmp", f), "params")
cross_estimation(dict)
```
即它只需加载`dict`并直接使用"主"函数`cross_estimation`。请别忘了定期清理`tmp`目录！你可以通过在 `runjob.jl`脚本的最后添加一行`rm(projectdir("_research", "tmp", f)`来做到这一点。

## 列出完成的模拟工作
继续 [模拟前的准备 & 运行模拟](@ref)一节的内容，我们现在想将所有这些模拟的结果收集到一个单一的DataFrame中`DataFrame`。我们将使用[`collect_results!`](@ref)函数实现此目的。
实际上非常简单！但是因为我们不想包含错误信息，所以必须将其列入黑名单：
```@example customizing
using DataFrames # this is necessary to access collect_results!
bl = ["error"]
res = collect_results!(datadir("results"); black_list = bl, subfolders = true)
```
我们还可以利用 [`collect_results!`](@ref) 的基本处理功能，来调用排除的 `"error"` 列，将其替换为其平均值：
```@example customizing
using Statistics: mean
special_list = [:avrg_error => data -> mean(data["error"])]
res = collect_results(
      datadir("results"),
      black_list = bl,
      special_list = special_list,
      subfolders = true
)

select!(res, Not(:path)) # don't show path this time
```
如您所见，我们使用了[`collect_results`](@ref)函数而不是它的in-place版本，因为已经存在一个包含所有已处理结果的`DataFrame`（因此所有内容都将被跳过）。

## 适配新的数据/参数
我们继续上面的例子。但现在我们需要用一些新的参数运行一些新的模拟，这些参数原本 _不存在_ 于旧模拟中... 但是，DrWatson说 “没问题！( ￣▽￣)σ ” 
让我们将这些新参数保存在一个不同的子文件夹中，以保持项目整洁有序：
```@example customizing
general_args_new = Dict(
    "model" => ["bocf"],
    "symmetry" => "radial",
    "symmetric_training" => [true, false],
    "N" => [100],
    "embedding" => [ #(γ, τ, r, c)
    (4, 5, 1, 0.34), (4, 6, 1, 0.28)]
)
```
如您所见，这里有两个在以前的模拟中不存在的参数，即`"symmetry", "symmetric_training"`。此外，在 _以前_ 模拟中存在的`"noise", "noisy_training"`参数在当前模拟中不存在。
没关系，让我们运行新的模拟：
```@example customizing
dicts = dict_list(general_args_new)
map(cross_estimation, dicts)

# load one of the files to be sure everything is ok:
filename = readdir(datadir("results", "bocf"))[1]
file = load(datadir("results", "bocf", filename))
```
好的，现在我们想要将这些新的运行结果 _添加_ 到已收集所有先前结果的现有数据框中。这很简单：
```@example customizing
res = collect_results!(datadir("results"); black_list = bl, subfolders = true)

select!(res, Not(:path)) # don't show path this time
```
所有的`missing`条目都被自动调整了 :)
## 定义带有限制的参数集
如上面的例子所示，对于每次模拟运行的输入参数集合相同的函数,可以使用基本字典来定义这些参数。然而，通常只有在包含另一个参数或具有特定值的情况下才应考虑某些参数或值。 [`@onlyif`](@ref)宏允许对值和参数设置此类限制。以下字典定义了用于遗传算法的值和参数:
```@example customizing
ga_parameters = Dict(
    :population_size => [20,50,100],
    :selection => ["roulette-selection", "SUS", "tournament-selection", "linear ranking"],
    :fitness_scaling => @onlyif(:selection in ("SUS", "roulette-selection"), collect(1.0:20.0)),
    :tournamet_size => @onlyif(:selection == "tournament-selection", collect(2:10)),
    :chromosome => [:A, @onlyif(begin
        size_constr = (:population_size <= 50)
        select_constr = (:selection != "SUS")
        size_constr && select_constr
    end, :B)])
```

```@example customizing
dicts = dict_list(ga_parameters)
length(dicts)
```

```@example customizing
dicts[1]
```
染色体类型的参数限制显示，可以使用任意的 Julia 表达式，这些表达式返回 `true` 或 `false`。
在这种情况下，首先评估并存储了关于种群大小和选择方法的条件。若两个条件都满足，则表达式仅返回 true，从而限制染色体类型`:B` 的使用。

由于 `@onlyif` 旨在与 [`dict_list`](@ref) 一起使用，它支持存储用于定义可能参数值的向量符号。
这是通过自动将每个`@onlyif`调用广播到`Vector` 参数来实现的，从而允许链接这些调用以组合条件。
因此，就结果而言，`@onlyif( :a == 2, [5, @onlyif(:b == 4, 6)])`等价于 `[@onlyif( :a == 2, 5), @onlyif(:a == 2 && :b == 4, 6)]`。

## 使用collect_results按名称过滤
在包含许多（例如 1000个）文件的文件夹上使用 [`collect_results`](@ref) 可能会明显变慢。为了加快速度，可以使用 `rinclude` 和 `rexclude` 关键字参数，它们都是 [正则表达式](https://docs.julialang.org/en/v1/manual/strings/#man-regex-literals) 的向量。返回的结果将匹配 `rinclude` 中的**任何**正则表达式且不匹配`rexclude` 中**任何**正则表达式的文件名。
```julia
df = collect_results(datadir("results"); rinclude=[r"a=1"])
# Only include results whose filename contains "a=1"

df = collect_results(datadir("results"); rexclude=[r"a=3"])
# Exclude any results whose filename contains "a=3"

df = collect_results(datadir("results"); rinclude=[r"a=1", r"b=5"], rexclude=[r"a=3"])
# Only include results whose filename contains "a=1" OR "b=5" and exclude any which contain "a=3"
```

## collect_results的高级用法
在您的工作中，您可能希望运行一个返回多个字段的函数，并将其包含在您的结果 `DataFrame` 中。
根据您要解决的问题，使用一个提取大多数或所有元数据的单个函数可能更有意义。针对这种情况，`DrWatson` 提供了另一种语法。为了简单起见，假设您的数据文件包含一个名为 `"manynumbers"` 的非常长的数组，而您关心的信息是其中的三个最大值。

一种实现方法是编写以下代码：
```julia
special_list = [
    :first  => data -> sort(data["manynumbers"])[1],
    :second => data -> sort(data["manynumbers"])[2],
    :third  => data -> sort(data["manynumbers"])[3],
    ]
```
很明显，对非常长的向量进行三次排序并没有必要，应该有更好的方法。更好的做法是：
```julia
function largestthree(data)
    sorted = sort(data["manynumbers"])
    return [:first  => sorted[1],
            :second => sorted[2],
            :third  => sorted[3]]
end

special_list = [largestthree,]
```

## 使用 `savename` 函数生成日志文件
当您的代码运行时间较长，抑或是代码在不同的机器（例如集群环境）上运行，那么生成日志文件就变得很重要。日志文件可以让您在程序仍在运行时查看其进度，或在稍后检查是否一切按计划进行。
```julia
using Dates

function logmessage(n, error)
    # current time
    time = Dates.format(now(UTC), dateformat"yyyy-mm-dd HH:MM:SS")

    # memory the process is using
    maxrss = "$(round(Sys.maxrss()/1048576, digits=2)) MiB"

    logdata = (;
        n, # iteration n
        error, # some super important progress update
        maxrss) # lastly the amount of memory being used

    println(savename(time, logdata; connector=" | ", equals=" = ", sort=false, digits=2))
end

function expensive_computation(N)

    for n = 1:N
        sleep(1) # heavy computation
        error = rand()/n # some super import progress update
        logmessage(n, error)
    end

end
```
这会产生易于阅读的*和*机器解析的输出日志。如果您最终拥有太多日志文件无法阅读，仍然可以使用 `parse_savename` 来帮助您。
```julia
julia> expensive_computation(5)
2021-05-19 19:20:25 | n = 1 | error = 0.65 | maxrss = 326.27 MiB
2021-05-19 19:20:26 | n = 2 | error = 0.48 | maxrss = 326.27 MiB
2021-05-19 19:20:27 | n = 3 | error = 0.08 | maxrss = 326.27 MiB
2021-05-19 19:20:28 | n = 4 | error = 0.11 | maxrss = 326.27 MiB
2021-05-19 19:20:29 | n = 5 | error = 0.15 | maxrss = 326.27 MiB
```

## 将项目的输入输出自动化推向极致

本节旨在展示如何充分利用  [`savename`](@ref) 和  [`produce_or_load`](@ref) 之间的协同作用，**将项目从输入到输出完全自动化，并尽可能消除重复的代码行**。请您先阅读 [自定义 savename](@ref customizing_savename)部分以获取相关信息。

实现完全自动化的重要因素是 [`produce_or_load`](@ref) 被设计成与[`savename`](@ref) 很好地协同工作。您可以按照以下步骤将项目中从输入到输出的工作流程完全自动化：

1. 定义一个自定义结构体，用来表示实验或模拟中的输入配置。
2. 扩展  [`savename`](@ref)  以适应自定义结构体。
3. 定义一个“主”函数，使得该函数以这种配置类型的实例作为输入，并返回实验或模拟的输出到字典变量（我们没有改变 Julia 中将文件保存为 `.jld2` 文件的“默认”方式。要以这种方式保存文件，您需要将数据存储在以 `String` 为键值的字典中）。
4. 您所有的输入输出脚本只需先定义输入配置类型，然后使用预定义的“主”函数调用  [`produce_or_load`](@ref) 函数即可（或者，该函数可以在内部调用 `produce_or_load` 并返回您感兴趣的特定内容）。

这种方法在“现实案例”中应用的一个例子是我们的论文[Effortless estimation of basins of attraction](https://arxiv.org/abs/2110.04358)，对应的github库是https://github.com/Datseris/EffortlessBasinsOfAttraction 。不用担心，您无需了解该领域即可阅读相关内容。关键是在我们的工作中需要对许多不同的动力系统运行某种模拟，而这些系统具有不同的参数、维度等。但它们确实有一个共同点：我们的输出始终来自同一个函数，即`basins_of_attraction`，使得上述函数 [`produce_or_load`](@ref)能被应用于工作流中。

因此，我们定义了一个名为 `BasinConfig` 的结构体，用来存储配置选项和系统参数。然后我们为它扩展了 `savename`函数。我们定义了一个函数 `produce_basins`来接收这个配置文件，相应地初始化一个动力系统，然后 **使用 `produce_or_load`** 生成结果。这确保了如果模拟已经存在，我们就不会再次运行它们。请注意，当您有相当多的参数和不同系统需要模拟时，很容易无意中运行相同的模拟两次，因为您“已经忘记运行了哪个”。我们提到的所有这些内容都可以在这个文件中找到：
https://github.com/Datseris/EffortlessBasinsOfAttraction/blob/master/src/produce_basins.jl

这样做的好处是什么？那就是：我们真正需要关心的脚本内容都非常简短：
```julia
using DrWatson
@quickactivate :EffortlessBasinsOfAttraction

a, b = 1.4, 0.3
p = @ntuple a b
system = :henon

basin_kwargs = (horizon_limit=100.0, mx_chk_fnd_att=30, mx_chk_lost=2)
Z = 201
xg = range(-1.5, 1.5; length = Z)
yg = range(-0.5, 0.5; length = Z)
grid = (xg, yg)

config = BasinConfig(; system, p, basin_kwargs, grid)
basins, attractors = produce_basins(config)
```
更重要的是，从不同脚本间唯一真正需要被“复制粘贴”的行是最后两行。所有其他行对于每个脚本都是唯一的。这种减少重复信息的复制粘贴成功最小化了工作流程的复杂性，并使错误更易于找到。