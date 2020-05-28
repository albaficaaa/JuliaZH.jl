# [类型转换和类型提升](@id conversion-and-promotion)

Julia 有一个提升系统，可以将数学运算符的参数提升为通用类型，如在前面章节中提到的[整数和浮点数](@ref)、[数学运算和初等函数](@ref)、[类型](@ref man-types)和[方法](@ref)。在本节中，我们将解释类型提升系统如何工作，以及如何将其扩展到新的类型，并将其应用于除内置数学运算符之外的其他函数。传统上，编程语言在参数的类型提升上分为两大阵营:

  * **内置数学类型和运算符的自动类型提升。**大多数语言中，内置数值类型，当作为带有中缀语法的算术运算符的操作数时，例如 `+`、`-`、`*` 和 `/` 将自动提升为通用类型，以产生预期的结果。举例来说，C、Java、Perl 和 Python，都将 `1 + 1.5` 的和作为浮点值 `2.5`，即使 `+` 的一个操作数是整数。这些系统非常方便且设计得足够精细，以至于它对于程序员来讲通常是不可见的：在编写这样的表达式时，几乎没有人有意识地想到这种类型提升，但编译器和解释器必须在相加前执行转换，因为整数和浮点值无法按原样相加。因此，这种自动类型转换的复杂规则不可避免地是这些语言的规范和实现的一部分。
     
     
     
     
     
     
     
     
     
  * **没有自动类型提升。**这个阵营包括 Ada 和 ML——非常「严格的」 静态类型语言。在这些语言中，每个类型转换都必须由程序员明确指定。因此，示例表达式 `1 + 1.5` 在 Ada 和 ML 中都会导致编译错误。相反地，必须编写 `real(1) + 1.5`，来在执行加法前将整数 `1` 显式转换为浮点值。然而，处处都显式转换是如此地不方便，以至于连 Ada 也有一定程度的自动类型转换：整数字面量被类型提升为预期的整数类型，浮点字面量同样被类型提升为适当的浮点类型。
     
     
     
     
     
     
     

在某种意义上，Julia 属于「无自动类型提升」类别：数学操作符只是具有特殊语法的函数，函数的参数永远不会自动转换。然而，人们可能会发现数学运算能应用于各种混合的参数类型，但这只是多态的多重分派的极端情况——这是 Julia 的分派和类型系统特别适合处理的情况。数学操作数的「自动」类型提升只是作为一个特殊的应用出现：Julia 带有预定义的数学运算符的 catch-all 分派规则，其在某些操作数类型的组合没有特定实现时调用。这些 catch-all 分派规则首先使用用户可定义的类型提升规则将所有操作数提升到一个通用的类型，然后针对结果值（现在已属于相同类型）调用相关运算符的特定实现。用户定义的类型可简单地加入这个类型提升系统，这需要先定义与其它类型进行相互类型转换的方法，接着提供一些类型提升规则来定义与其它类型混合时应该提升到什么类型。

## 类型转换

The standard way to obtain a value of a certain type `T` is to call the type's constructor, `T(x)`.
However, there are cases where it's convenient to convert a value from one type to another
without the programmer asking for it explicitly.
One example is assigning a value into an array: if `A` is a `Vector{Float64}`, the expression
`A[1] = 2` should work by automatically converting the `2` from `Int` to `Float64`, and
storing the result in the array.
This is done via the [`convert`](@ref) function.

`convert` 函数通常接受两个参数：第一个是类型对象，第二个是需要转换为该类型的值。返回的是已转换后的值。理解这个函数最简单的办法就是尝试：

```jldoctest
julia> x = 12
12

julia> typeof(x)
Int64

julia> convert(UInt8, x)
0x0c

julia> typeof(ans)
UInt8

julia> convert(AbstractFloat, x)
12.0

julia> typeof(ans)
Float64

julia> a = Any[1 2 3; 4 5 6]
2×3 Array{Any,2}:
 1  2  3
 4  5  6

julia> convert(Array{Float64}, a)
2×3 Array{Float64,2}:
 1.0  2.0  3.0
 4.0  5.0  6.0
```

类型转换并不总是可行的，有时 `convert` 函数并不知道该如何执行所请求的类型转换就会抛出 no method error 错误。例如下例：

```jldoctest
julia> convert(AbstractFloat, "foo")
ERROR: MethodError: Cannot `convert` an object of type String to an object of type AbstractFloat
[...]
```

Some languages consider parsing strings as numbers or formatting numbers as strings to be conversions
(many dynamic languages will even perform conversion for you automatically), however Julia does
not: even though some strings can be parsed as numbers, most strings are not valid representations
of numbers, and only a very limited subset of them are. Therefore in Julia the dedicated [`parse`](@ref)
function must be used to perform this operation, making it more explicit.

### 什么时候使用 `convert` 函数?

构造以下语言结构时需要调用 `convert` 函数：

  * 对一个数组赋值会转换为数组元素的类型。
  * 对一个对象的字段赋值会转换为已声明的字段类型。
  * Constructing an object with [`new`](@ref) converts to the object's declared field types.
  * 对已声明类型的变量赋值（例如 `local x::T`）会转换为该类型。
  * 已声明返回类型的函数会转换其返回值为该类型。
  * Passing a value to [`ccall`](@ref) converts it to the corresponding argument type.

### 类型转换与构造

注意到 `convert(T, x)` 的行为似乎与 `T(x)` 几乎相同，它的确通常是这样。但是，有一个关键的语义差别：因为 `convert` 能被隐式调用，所以它的方法仅限于被认为是「安全」或「意料之内」的情况。`convert` 只会在表示事物的相同基本种类的类型之间进行转换（例如，不同的数字表示和不同的字符串编码）。它通常也是无损的；将值转换为其它类型并再次转换回去应该产生完全相同的值。

这是四种一般的构造函数与 `convert` 不同的情况：

#### 与其参数类型无关的类型的构造函数

一些构造函数没有体现「转换」的概念。例如，`Timer(2)` 创建一个时长 2 秒的定时器，它实际上并不是从整数到定时器的「转换」。

#### 可变的集合

如果 `x` 类型已经为 `T`，`convert(T, x)` 应该返回原本的 `x`。相反地，如果 `T` 是一个可变的集合类型，那么 `T(x)` 应该总是创建一个新的集合（从 `x` 复制元素）。

#### 封装器类型

对于某些「封装」其它值的类型，构造函数可能会将其参数封装在一个新对象中，即使它已经是所请求的类型。例如，用 `Some(x)` 表示封装了一个 `x` 值（在上下文中，其结果可能是一个 `Some` 或 `nothing`）。但是，`x` 本身可能是对象 `Some(y)`，在这种情况下，结果为 `Some(Some(y))`，封装了两层。然而，`convert(Some, x)` 只会返回 `x`，因为它已经是 `Some` 的实例了。

#### 不返回自身类型的实例的构造函数

在*极少见*的情况下，构造函数 `T(x)` 返回一个类型不为 `T` 的对象是有意义的。如果封装器类型是它自身的反转（例如 `Flip(Flip(x)) === x`），或者在重构库时为了支持某个旧的调用语法以实现向后兼容，则可能发生这种情况。但是，`convert(T, x)` 应该总是返回一个类型为 `T` 的值。

### 定义新的类型转换

在定义新类型时，最初创建它的所有方法都应定义为构造函数。如果隐式类型转换很明显是有用的，并且某些构造函数满足上面的「安全」标准，那么可以考虑添加 `convert` 方法。这些方法通常非常简单，因为它们只需要调用适当的构造函数。此类定义可能会像这样：

```julia
convert(::Type{MyType}, x) = MyType(x)
```

此方法的第一个参数的类型是[单态类型](@ref man-singleton-types) `Type{MyType}`，其唯一实例是 `MyType`。因此，此方法仅在第一个参数是类型值 `MyType` 时才被调用。请注意第一个参数使用的语法：在 `::` 符号之前省略了参数名，只是给出了参数类型。这是 Julia 中用于函数参数的语法，该参数的类型已经指定，但其值无需通过名称引用。在此例中，由于参数类型是单态类型，我们已经知道其值而无需引用参数名称。

某些抽象类型的所有实例默认都被认为是「足够相似的」，在 Julia Base 中也提供了通用的 `convert` 定义。例如，这个定义声明通过调用单参数构造函数将任何 `Number` 类型 `convert` 为其它任何 `Number` 类型是有效的：

```julia
convert(::Type{T}, x::Number) where {T<:Number} = T(x)
```

这意味着新的 `Number` 类型只需要定义构造函数，因为此定义将为它们处理 `convert`。在参数已经是所请求的类型的情况下，用恒同变换来处理 `convert`。

```julia
convert(::Type{T}, x::T) where {T<:Number} = x
```

Similar definitions exist for `AbstractString`, [`AbstractArray`](@ref), and [`AbstractDict`](@ref).

## 类型提升

类型提升是指将一组混合类型的值转换为单个通用类型。尽管不是绝对必要的，但一般暗示被转换的值的通用类型可以忠实地表示所有原始值。此意义下，术语「类型提升」是合适的，因为值被转换为「更大」的类型——即能用一个通用类型表示所有输入值的类型。但重要的是，不要将它与面向对象（结构）超类或 Julia 的抽象超类型混淆：类型提升与类型层次结构无关，而与备选的表示之间的转换有关。例如，尽管每个 [`Int32`](@ref) 值可以表示为 [`Float64`](@ref) 值，但 `Int32` 不是 `Float64` 的子类型。


Promotion to a common "greater" type is performed in Julia by the [`promote`](@ref) function, which takes
any number of arguments, and returns a tuple of the same number of values, converted to a common
type, or throws an exception if promotion is not possible. The most common use case for promotion
is to convert numeric arguments to a common type:

```jldoctest
julia> promote(1, 2.5)
(1.0, 2.5)

julia> promote(1, 2.5, 3)
(1.0, 2.5, 3.0)

julia> promote(2, 3//4)
(2//1, 3//4)

julia> promote(1, 2.5, 3, 3//4)
(1.0, 2.5, 3.0, 0.75)

julia> promote(1.5, im)
(1.5 + 0.0im, 0.0 + 1.0im)

julia> promote(1 + 2im, 3//4)
(1//1 + 2//1*im, 3//4 + 0//1*im)
```

浮点值被提升为最大的浮点参数类型。整数值会被提升为本机机器字大小或最大的整数参数类型中较大的一个。整数和浮点值的混合会被提升为一个足以包含所有值的浮点类型。与有理数混合的整数会被提升有理数。与浮点数混合的有理数会被提升为浮点数。与实数值混合的复数值会被提升为合适类型的复数值。

这就是使用类型提升的全部内容。剩下的只是聪明的应用，最典型的「聪明」应用是数值操作（如 `+`、`-`、`*` 和 `/`）的 catch-all 方法的定义。以下是在 [`promotion.jl`](https://github.com/JuliaLang/julia/blob/master/base/promotion.jl) 中给出的几个 catch-all 方法的定义：

```julia
+(x::Number, y::Number) = +(promote(x,y)...)
-(x::Number, y::Number) = -(promote(x,y)...)
*(x::Number, y::Number) = *(promote(x,y)...)
/(x::Number, y::Number) = /(promote(x,y)...)
```

这些方法的定义表明，如果没有更特殊的规则来加、减、乘及除一对数值，则将这些值提升为通用类型并再试一次。这就是它的全部内容：在其它任何地方都不需要为数值操作担心到通用数值类型的类型提升——它会自动进行。许多算术和数学函数的 catch-all 类型提升方法的定义在 [`promotion.jl`](https://github.com/JuliaLang/julia/blob/master/base/promotion.jl) 中，但除此之外，Julia Base 中几乎不再需要调用 `promote`。`promote` 最常用于外部构造方法中，为了更方便，可允许使用混合类型的构造函数调用委托给一个内部构造函数，并将字段提升为适当的通用类型。例如，回想一下，[`rational.jl`](https://github.com/JuliaLang/julia/blob/master/base/rational.jl) 提供了以下外部构造方法：

```julia
Rational(n::Integer, d::Integer) = Rational(promote(n,d)...)
```

这允许像下面这样的调用正常工作：

```jldoctest
julia> Rational(Int8(15),Int32(-5))
-3//1

julia> typeof(ans)
Rational{Int32}
```

对于大多数用户定义的类型，最好要求程序员明确地向构造函数提供期待的类型，但有时，尤其是对于数值问题，自动进行类型提升会很方便。

### 定义类型提升规则

Although one could, in principle, define methods for the `promote` function directly, this would
require many redundant definitions for all possible permutations of argument types. Instead, the
behavior of `promote` is defined in terms of an auxiliary function called [`promote_rule`](@ref), which
one can provide methods for. The `promote_rule` function takes a pair of type objects and returns
another type object, such that instances of the argument types will be promoted to the returned
type. Thus, by defining the rule:

```julia
promote_rule(::Type{Float64}, ::Type{Float32}) = Float64
```

声明当同时类型提升 64 位和 32 位浮点值时，它们应该被类型提升为 64 位浮点数。但是，提升类型不需要是参数类型之一；在 Julia Base 中有以下类型提升规则：

```julia
promote_rule(::Type{BigInt}, ::Type{Float64}) = BigFloat
promote_rule(::Type{BigInt}, ::Type{Int8}) = BigInt
```

在后一种情况下，输出类型是 [`BigInt`](@ref)，因为 `BigInt` 是唯一一个足以容纳任意精度整数运算结果的类型。还要注意，不需要同时定义 `promote_rule(::Type{A}, ::Type{B})` 和 `promote_rule(::Type{B}, ::Type{A})`——对称性隐含在类型提升过程中使用 `promote_rule` 的方式。

The `promote_rule` function is used as a building block to define a second function called [`promote_type`](@ref),
which, given any number of type objects, returns the common type to which those values, as arguments
to `promote` should be promoted. Thus, if one wants to know, in absence of actual values, what
type a collection of values of certain types would promote to, one can use `promote_type`:

```jldoctest
julia> promote_type(Int8, Int64)
Int64
```

在内部，`promote_type` 在 `promote` 中用于确定参数值应被转换为什么类型以便进行类型提升。但是，它本身可能是有用的。好奇的读者可以阅读 [`promotion.jl`](https://github.com/JuliaLang/julia/blob/master/base/promotion.jl)，该文件用大概 35 行定义了完整的类型提升规则。

### 案例研究：有理数的类型提升

最后，我们来完成关于 Julia 的有理数类型的案例研究，该案例通过以下类型提升规则相对复杂地使用了类型提升机制：

```julia
promote_rule(::Type{Rational{T}}, ::Type{S}) where {T<:Integer,S<:Integer} = Rational{promote_type(T,S)}
promote_rule(::Type{Rational{T}}, ::Type{Rational{S}}) where {T<:Integer,S<:Integer} = Rational{promote_type(T,S)}
promote_rule(::Type{Rational{T}}, ::Type{S}) where {T<:Integer,S<:AbstractFloat} = promote_type(T,S)
```

第一条规则说，使用其它整数类型类型提升有理数类型会得到个有理数类型，其分子/分母类型是使用其它整数类型提升该有理数分子/分母类型的结果。第二条规则将相同的逻辑应用于两种不同的有理数类型，它们进行类型提升会得到有理数类型，其分子/分母类型是它们各自的分子/分母类型进行提升的结果。第三个也是最后一个规则规定，使用浮点数类型提升有理数类型与使用该浮点数类型提升其分子/分母类型会产生相同的类型。

这一小部分的类型提升规则，连同该类型的构造函数和数字的默认 `convert` 方法，便足以使有理数与 Julia 的其它数值类型——整数、浮点数和复数——完全自然地互操作。通过以相同的方式提供类型转换方法和类型提升规则，任何用户定义的数值类型都可像 Julia 的预定义数值类型一样自然地进行互操作。