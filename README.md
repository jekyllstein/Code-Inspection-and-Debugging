# Code-Inspection-and-Debugging
## References
- https://github.com/JuliaDebug has all relevant debugging packages
- https://github.com/JuliaDebug/Cthulhu.jl Cthulhu package: Cthulhu can help you debug type inference issues by recursively showing the code_typed output until you find the exact point where inference gave up, messed up, or did something unexpected.
- https://github.com/JuliaDebug/Debugger.jl Debugger.jl package: Allows for manipulating program execution, such as stepping in and out of functions, line stepping, showing local variables, setting breakpoints and evaluating code in the context of functions.

## Type inference debugging with Cthulhu.jl
### Basic Functionality
- Tool for debugging type instabilities with an alternative workflow to:
       - @code_typed
       - search for call(g, ...)::Any
       - code_typed(g, ...)
       - repeat through many layers before finding the problem
- Not a debugger, operates on typed IR not lowered IR, intended for use on abstractions who's type inference you're curious about, not runtime values
- Helpful for cases where the instability only occurs in context, not when the call is inspected on its own
- To begin use the `@descend` macro to explore typed IR which will reveal a menu of options
```julia
julia> h(A, B) = A .+ B .* A
h (generic function with 1 method)
julia> @descend h(zeros(3), zeros(3, 3))  
...
│    ││││││ %266 = %new(Base.OneTo{Int64}, %265)::Base.OneTo{Int64}
│    │││││└
│    │││││ %267 = (Core.tuple)(%263, %266)::Tuple{Base.OneTo{Int64},Base.OneTo{Int64}}
│    │││└└
│    │││        invoke Base.Broadcast.throwdm(%267::Tuple{Base.OneTo{Int64},Base.OneTo{Int64}}, %76::Tuple{Base.OneTo{Int64},Base.OneTo{Int64}})::Union{}
└────│││        $(Expr(:unreachable))::Union{}
     │││ @ broadcast.jl:797 within `copyto!'
100 ┄│││        goto #101
     ││└
101 ─││        goto #102
     │└
102 ─│        goto #103
     └
103 ─        return %81
)
Select a call to descend into or ↩ to ascend. [q]uit.
Toggles: [o]ptimize, [w]arn, [d]ebuginfo, [s]yntax highlight for LLVM/Native.
Show: [L]LVM IR, [N]ative code
Advanced: dump [P]arams cache.

 • %268  = invoke throwdm(::Tuple{Base.OneTo{Int64},Base.OneTo{Int64}},::Tuple{Base.OneTo{Int64},Base.OneTo{Int64}})::Union{}
   ↩
```
- hit `o` to enter unoptimized view and inspect what h does before compiler optimizations
```julia
│ ─ %-1  = invoke h(::Array{Float64,1},::Array{Float64,2})::Array{Float64,2}
CodeInfo(
    @ REPL[102]:1 within `h'
1 ─ %1 = (Base.broadcasted)(Main.:*, B, A)::Base.Broadcast.Broadcasted{Base.Broadcast.DefaultArrayStyle{2},Nothing,typeof(*),Tuple{Array{Float64,2},Array{Float64,1}}}
│   %2 = (Base.broadcasted)(Main.:+, A, %1)::Base.Broadcast.Broadcasted{Base.Broadcast.DefaultArrayStyle{2},Nothing,typeof(+),Tuple{Array{Float64,1},Base.Broadcast.Broadcasted{Base.Broadcast.DefaultArrayStyle{2},Nothing,typeof(*),Tuple{Array{Float64,2},Array{Float64,1}}}}}
│   %3 = (Base.materialize)(%2)::Array{Float64,2}
└──      return %3
)
Select a call to descend into or ↩ to ascend. [q]uit.
Toggles: [o]ptimize, [w]arn, [d]ebuginfo, [s]yntax highlight for LLVM/Native.
Show: [L]LVM IR, [N]ative code
Advanced: dump [P]arams cache.

 • %1  = invoke broadcasted(::typeof(*),::Array{Float64,2},::Array{Float64,1})
   %2  = invoke broadcasted(::typeof(+){…},::Array{…},::Base.Broadcast.Broadcasted{…})
   %3  = invoke materialize(::Base.Broadcast.Broadcasted{…})::Array{Float64,2}
   ↩
```
- %1, %2, and %3 here are all steps that get optimized away
- can descend into materialize and then copy where we see type signature `Union{}` which means the compiler believes this will never execute
- go into copyto! twice due to recursion and then actually see how broadcast is executed, can turn on optimization and debugging to see inlining structure and hit `L` to view LLVM IR, hit `q` any time to exit

### Type Instability Example
```julia
julia> struct Interval <: Number

           a::Float64

           b::Float64

       end

julia> Base.eltype(::Interval) = Float64

julia> contains(i::Interval, x::Float64) = i.a <= x <= i.b
contains (generic function with 1 method)

julia> f(I::Interval, xs::Vector{Float64}) = contains.(I, xs)
f (generic function with 4 methods)

julia> f(Interval(0.5, 0.6), rand(4))
4-element BitArray{1}:
  true
 false
 false
 false
```
- all f does is broadcast the contains function across xs and we'd expect this to be static code with the return type always being a BitArray
- if we check this with @code_warntype, we see something weird

```julia
julia> @code_warntype f(Interval(0.5, 0.6), rand(4))
...
23 ─ %87 = invoke Base.Broadcast.copyto_nonleaf!(%51::BitArray{1}, %15::Base.Broadcast.Broadcasted{Base.Broadcast.DefaultArrayStyle{1},Tuple{Base.OneTo{Int64}},typeof(contains),Tuple{Interval,Base.Broadcast.Extruded{Array{Float64,1},Tuple{Bool},Tuple{Int64}}}}, %4::Base.OneTo{Int64}, %22::Int64, 1::Int64)::BitArray{1}
└───       goto #24
24 ┄ %89 = φ (#5 => %24, #23 => %87)::Union{BitArray{1}, Array{Union{},1}}
└───       goto #25
25 ─       return %89
```
- function seems to return either a BitArray or an Array of empty Union, but that type signature appears nowhere in the body
- try @descend instead and we see that all the calls only produce BitArrays, look at unoptimized instead
- in unoptimized code, see that materialize can return Array{Union{}, 1}
- step again into copy, and we begin to see evidence of the problem such as
```
   %13 = (Base.getproperty)(bc, :args)::Tuple{Interval,Array{Float64,1}}
│         (ElType = (Base.Broadcast.combine_eltypes)(%12, %13))
│   @ broadcast.jl:771 within `copy'
│   %15 = Base.isconcretetype::Core.Compiler.Const(isconcretetype, false)
│   %16 = (%15)(ElType::Core.Compiler.Const(Union{}, false))::Core.Compiler.Const(false, true)
```
- for some reason combine_eltypes cannot infer a concrete type which defaults to slow behavior like unoptimized loops, can step into combine_eltypes as well
```
│ ─ %-1  = invoke combine_eltypes(::typeof(contains),::Tuple{Interval,Array{Float64,1}})::Core.Compiler.Const(Union{}, false)
Body::Core.Compiler.Const(Union{}, false)
    @ broadcast.jl:627 within `combine_eltypes'
1 ─ %1 = Base._return_type::Core.Compiler.Const(Core.Compiler.return_type, false)
│   %2 = (Base.Broadcast.eltypes)(args)::Core.Compiler.Const(Tuple{Float64,Float64}, false)
│   %3 = (%1)(f, %2)::Core.Compiler.Const(Union{}, false)
└──      return %3
```
- we see a `Tuple{Float64, Float64}` which seems to match the interval, but when we call `Base._return_type` on that the infered result is Union{}
- an attempt to step into `return_type` leads to an error
```
   %2  = invoke eltypes(::Tuple{Interval,Array{Float64,1}})::Core.Compiler.Const(Tuple{Float64,Float64}, false)
 • %3  = return_type < #contains(::Float64,::Float64)::Core.Compiler.Const(Union{}, false) >
   ↩
┌ Error: MethodInstance extraction failed
│   ci.sig = Tuple{typeof(contains),Float64,Float64}
│   ci.rt = Core.Compiler.Const(Union{}, false)
└ @ Cthulhu ~/.julia/packages/Cthulhu/PyZJN/src/callsite.jl:21
```
- so type inference has failed on the contains call and that is the source of Union{} but why?
- the method that does not exist here is `contains(::Float64, ::Float64)` and since `return_type` isn't allowed to fail it defaults to Union{}
- because we defined `Base.eltype(::Interval) = Float64` we enter inference with Tuple{Float64, Float64} on contains
- if we start a new REPL and do not define Base.eltype then there is no type instability


## Runtime Debugging with Debugger.jl
- JuliaInterpreter.jl provides the backend for debugging tools
- JuliaInterpreter is able to evaluate Julia’s lowered representation statement-by-statement, and thus serves as the foundation for inspecting and manipulating intermediate results.
- Debugger.jl offers the most powerful control over stepping and has some capabilities that none of the other interfaces offer (e.g., very fine-grained control over stepping, the ability to execute the generator of generated functions, etc.), so it should be your go-to choice for particularly difficult cases.
- prepend a function call with the @enter macro
```julia
julia> function g(a, b)
       c = a+b
       c^2
       end
g (generic function with 1 method)

julia> @enter g(1, 2)
In g(a, b) at REPL[83]:2
 1  function g(a, b)
>2  c = a+b
 3  c^2
 4  end

About to run: (+)(1, 2)
1|debug>
 ```
- `1:debug>` is the debugger repl that we can use to do stepping and inspection
- `n` will step to the next line of the function until the final return result
```julia
1|debug> n
In g(a, b) at REPL[83]:2
 1  function g(a, b)
 2  c = a+b
>3  c^2
 4  end

About to run: (Core.apply_type)(Val, 2)
1|debug> n
In g(a, b) at REPL[83]:2
 1  function g(a, b)
 2  c = a+b
>3  c^2
 4  end

About to run: return 9
1|debug> n
9
```
- Once the final return value is reached, stepping forward at all will return to the usual REPL.  To exit debugging mode at any time hit `q`
- the interface shows the current line at each step as well as what call is about to run
- using the `fr` command we can also inspect the "frame" which shows all the local variables and their values at the point, let's inspect the frame right before the squaring takes place.
```julia
About to run: (Core.apply_type)(Val, 2)
1|debug> fr
[1] g(a, b) at REPL[83]:2
  | a::Int64 = 1
  | b::Int64 = 2
  | c::Int64 = 3
```
- another feature of debugger is the ability to change the repl mode back into "Julia REPL mode" where we can manipulate local variables.  This mode can be activated by typing \` in the debug REPL.  Let's use this to change the stored value of `a` before it is summed.
```julia
1|debug> fr
[1] g(a, b) at REPL[83]:2
  | a::Int64 = 1
  | b::Int64 = 2
1|julia> a = 2
2

1|debug> fr
[1] g(a, b) at REPL[83]:2
  | a::Int64 = 2
  | b::Int64 = 2
1|debug> n
In g(a, b) at REPL[83]:2
 1  function g(a, b)
 2  c = a+b
>3  c^2
 4  end

About to run: (Core.apply_type)(Val, 2)
1|debug> fr
[1] g(a, b) at REPL[83]:2
  | a::Int64 = 2
  | b::Int64 = 2
  | c::Int64 = 4
1|debug> n
In g(a, b) at REPL[83]:2
 1  function g(a, b)
 2  c = a+b
>3  c^2
 4  end

About to run: return 16
```
- Notice that the final result is now 16 and the Julia REPL mode is indicated by 1|julia>
- Rather than using `n` to step by whole lines, we can step into calls with `s`, for example let's step into the first call for sum.
```julia
About to run: (+)(1, 2)
1|debug> s
In +(x, y) at int.jl:53
>53  (+)(x::T, y::T) where {T<:BitInteger} = add_int(x, y)

About to run: (add_int)(1, 2)
1|debug> s
In +(x, y) at int.jl:53
>53  (+)(x::T, y::T) where {T<:BitInteger} = add_int(x, y)

About to run: return 3
1|debug> fr
[1] +(x, y) at int.jl:53
  | x::Int64 = 1
  | y::Int64 = 2
  | T::DataType = Int64
```
- Inspecting the frame at this point reveals a new local variable, T, that stores the type of the numbers being added
- Let's see how this varies when we pass a mixture of types of types to the function and step through all the promotion steps
- Can also do this for the squarring step and the call steps are much more complicated

