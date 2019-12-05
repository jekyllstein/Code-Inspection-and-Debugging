# Code-Inspection-and-Debugging
## References
- https://github.com/JuliaDebug has all relevant debugging packages
- https://github.com/JuliaDebug/Cthulhu.jl Cthulhu package: Cthulhu can help you debug type inference issues by recursively showing the code_typed output until you find the exact point where inference gave up, messed up, or did something unexpected.
- https://github.com/JuliaDebug/Debugger.jl Debugger.jl package: Allows for manipulating program execution, such as stepping in and out of functions, line stepping, showing local variables, setting breakpoints and evaluating code in the context of functions.

## Type inference debugging with Cthulhu.jl
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
- 

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

