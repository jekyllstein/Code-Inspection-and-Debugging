# Code-Inspection-and-Debugging

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

