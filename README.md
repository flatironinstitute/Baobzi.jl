# Baobzi.jl
Wrapper library for Baobzi interpolator library

TLDR: Baobzi takes a function and represents adatively it as a tree of polynomial
interpolants. This is _very_ fast and _very_ accurate. Basically it allows you to precompute a
"heavy" function into a very good approximation of that function that can be orders of
magnitude faster. Operations on 1D functions are typically ~100-200 million evaluations per
second, depending on hardware. I.e. functions can often evaluate faster than some fancy
implementations of functions like `log`, and can be orders of magnitude faster than special function evaluations.

Unfortunately, Baobzi is not a pure Julia implementation. The core library is written in `c++`
and has with `c` bindings as a public API. Since `Baobzi` needs to call the function you want
to approximate, this presents some limitations with Julia due to mutability and type
constraints. Basically you need to make your "callback" function C callable (`double
myfunc(double* x))`.  That or generate the function object in another language and call it in
julia. This is probably best shown by example. All functions you want to fit need a signature
identical to the `testfunc` below.

Simple 2d example for the function `z = x*y`
```julia
import Baobzi

function testfunc(xp::Ptr{Float64})::Cdouble
    x = unsafe_load(xp, 1)
    y = unsafe_load(xp, 2)
    return x * y
end

center = [0.0, 0.0]             # center of our function's rectangular domain
hl = [1.0, 1.0]                 # half widths of the functions rectangular domain
test_point = [0.25, 0.25]       # just a point we want to evaluate at
dim = 2                         # number of independent variables in our function (x, y)
order = 6                       # order of polynomial fit
tol = 1E-10                     # requested precision of fit
output_file = "simple2d.baobzi" # place to store the file
split_multi_eval = 0            # Whether to split evaluation into two steps or not. Has mild performance implications

# Fit our function
func_approx = Baobzi.init(testfunc, dim, order, center, hl, tol, 0)
# Print some stats about the fit procedure
Baobzi.stats(func_approx)
# Compare result
println(Baobzi.eval(func_approx, test_point) - testfunc(pointer(test_point)))

# Save our function and then delete it
Baobzi.save(func_approx, output_file)
Baobzi.free(func_approx)

# Restore function and compare
func_approx = Baobzi.restore(output_file)
println(Baobzi.eval(func_approx, test_point) - testfunc(pointer(test_point)))

# Evaluate function at many points in our domain
points = 2.0 * (rand(Float64, 2000000)) .- 1.0
z = Baobzi.eval_multi(func_approx, points)
```
