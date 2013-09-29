# Cartesian.jl

Fast multidimensional algorithms for the Julia language.

This package provides macros that currently appear to be the most performant way
to implement numerous multidimensional algorithms in Julia.

## Caution

In practice, `Cartesian` effectively introduces a separate "dialect" of
Julia. There is reason to hope that the main language will eventually
support much of this functionality, and if/when that happens some or all of this
should become obsolete. In the meantime, this package appears to be the most
powerful way to write efficient multidimensional algorithms.

## Installation

Install with the package manager, `Pkg.add("Cartesian")`.

## Principles of usage

Most macros in this package work like this:
```
@nloops 3 i A begin
    s += @nref 3 A i
end
```
which generates the following code:
```
for i3 = 1:size(A,3)
    for i2 = 1:size(A,2)
        for i1 = 1:size(A,1)
            s += A[i1,i2,i3]
        end
    end
end
```
The syntax of `@nloops` is as follows:

- The first argument must be an integer (_not_ a variable) specifying the number
of loops.
- The second argument is the symbol-prefix used for the iterator variable. Here
we used `i`, and variables `i1, i2, i3` were generated.
- The third argument specifies the range for each iterator variable. If you use
a variable (symbol) here, it's taken as `1:size(A,dim)`. More flexibly, you can
use the anonymous-function expression syntax described below.
- The last argument is the body of the loop. Here, that's what appears between
the `begin...end`.

`@nref` follows a similar pattern, generating `A[i1,i2,i3]` from `@nref 3 A i`.
The general practice is to read from left to right, which is why
`@nloops` is `@nloops 3 i A expr` (as in `for i2 = 1:size(A,2)`, where `i2` is
to the left and the range is to the right) whereas `@nref` is `@nref 3 A i` (as
in `A[i1,i2,i3]`, where the array comes first).

There are two important additional general points described below.

#### Supplying the dimensionality from functions

The first argument to both of these macros is the dimensionality, which must be
an integer. When you're writing a function that you intend to work in multiple
dimensions, this may not be something you want to hard-code. Fortunately, it's
straightforward to use an `@eval` construct, like this:

```
for N = 1:4
    @eval begin
        function laplacian{T}(A::Array{T,$N})
            B = similar(A)
            @nloops $N i A begin
                ...
            end
        end
    end
end
```

This would generate versions of `laplacian` for dimensions 1 through 4. While
it's somewhat more awkward, you can generate truly arbitrary-dimension functions
using a wrapper that keeps track of whether it has already compiled a version of
the function for different dimensionalities and data types:

```
let _mysum_defined = Dict{Any, Bool}()
global mysum
function mysum{T,N}(A::StridedArray{T,N})
    def = get(_mysum_defined, typeof(A), false)
    if !def
        ex = quote
            function _mysum{T}(A::StridedArray{T,$N})
                s = zero(T)
                @nloops $N i A begin
                    s += @nref $N A i
                end
                s
            end
        end
        eval(current_module(), ex)
        _mysum_defined[typeof(A)] = true
    end
    _mysum(A)
end
end
```

In addition to being longer than the first version, there's a (small)
performance price for checking the dictionary.

#### Anonymous-function expressions as macro arguments

Perhaps the single most powerful feature in `Cartesian` is the ability to supply
anonymous-function expressions to many macros. Let's consider implementing the
`laplacian` function mentioned above. The (discrete) laplacian of a two
dimensional array would be calculated as

```
lap[i,j] = A[i+1,j] + A[i-1,j] + A[i,j+1] + A[i,j-1] - 4A[i,j]
```

One obvious issue with this formula is how to handle the edges, where `A[i-1,j]`
might not exist. There are several strategies we can pursue, some of which will
be described below. As a first illustration of anonymous-function expressions,
let's just "punt" on the edges and just skip the edges. In 2d we might write
such code as

```
for i2 = 2:size(A,2)-1
    for i1 = 2:size(A,1)-1
        lap[i1,i2] = ...
    end
end
```

In `Cartesian` this can be written in the following way:

```
@nloops 2 i d->(2:size(A,d)-1) begin
    (@nref 2 lap i) = ...
end
```

Note here that the range argument is being supplied as an anonymous function. A
key point is that this anonymous function is _evaluated when the macro runs_.
(Whatever symbol appears as the variable of the anonymous function, here `d`, is
looped over the dimensionality.) Effectively, this expression gets _inlined_,
and hence generates exactly the code above with no extra runtime overhead.

There is an important bit of extra syntax associated with these expressions: the
expression `i_d`, for `d=3`, is translated into `i3`. Suppose we have two sets
of variables, a "main" group of indices `i1`, `i2`, and `i3`, and an "offset" group
of indices `j1`, `j2`, and `j3`. Then the expression

```
@nref 3 A d->(i_d+j_d)
```

gets translated into

```
A[i1+j1, i2+j2, i3+j3]
```

## A complete example: implementing `imfilter`

With this, we have enough machinery to implement a simple multidimensional
function `imfilter`, which computes the correlation (like a convolution) between
an array `A` and a filtering kernel `kern`. We're going to require that
`kernel` has odd-valued sizes along each dimension, say of size `2*w[d]+1`, and
suppose that there is a function `padarray` that pads `A` with width `w[d]`
along each edge in dimension `d`, using whatever boundary conditions the user
desires.

A complete implementation of `imfilter` is:

```
for N = 1:5
    @eval begin
        function imfilter{T}(A::Array{T,$N}, kern::Array{T,$N}, boundaryargs...)
            w = [div(size(kern, d), 2) for d = 1:$N]
            for i = 1:$N
                if size(kern, d) != 2*w[d]+1
                    error("kernel must have odd size in each dimension")
                end
            end
            Apad = padarray(A, w, boundaryargs...)
            B = similar(A)
            @nloops $N i A begin
                # Compute the filtered value
                tmp = zero(T)
                @nloops $N j kern begin
                    tmp += (@nref $N Apad d->(i_d+j_d-1))*(@nref $N kern j)
                end
                # Store the result
                (@nref $N B i) = tmp     # note the ()
            end
            B
        end
    end
end
```

The line

```
tmp += (@nref $N Apad d->(i_d+j_d-1))*(@nref $N kern j)
```

gets translated into

```
tmp += Apad[i1+j1-1, i2+j2-1, ...] * kern[j1, j2, ...]
```

We also note that the assignment to `B` requires parenthesis around the `@nref`,
because otherwise the expression `i = tmp` would be passed as the final argument
of the `@nref` macro.


## A complete example: implementing `laplacian`

We could implement the laplacian with `imfilter`, but that would be quite
wasteful: we don't need the "corners" of the 3x3x... grid, just its edges and
center. Consequently, we can write a considerably faster algorithm. Implementing
this algorithm will further illustrate the flexibility of anonymous-function
range expressions as well as another key macro, `@nexprs`.

In two dimensions, we'll generate the following code:

```
function laplacian{T}(A::Array{T,$N})
    B = similar(A)
    @nloops $N i A begin
        tmp = zero(T)
        tmp += i1 < size(A,1) ? A[i1+1,i2] : A[i1,i2]
        tmp += i2 < size(A,2) ? A[i1,i2+1] : A[i1,i2]
        tmp += i1 > 1 ? A[i1-1,i2] : A[i1,i2]
        tmp += i2 > 1 ? A[i1,i2-1] : A[i1,i2]
        B[i1,i2] = tmp - 4*A[i1,i2]
    end
    B
end
```
This uses "replicating boundary conditions" to handle the edges gracefully.

To generate those terms with all but one of the indices unaltered, we'll use
an anonymous function like this:

```
d2->(d2 == d1) ? i_d2+1 : i_d2
```

which shifts by 1 only when `d2 == d1`. We'll use the macro `@nexprs`
(documented below) to generate the `N` expressions we need. Here is the complete
implementation for dimensions 1 through 5:

```
for N = 1:5
    @eval begin
        function laplacian{T}(A::Array{T,$N})
            B = similar(A)
            @nloops $N i A begin
                tmp = zero(T)
                # Do the shift by +1.
                @nexprs $N d1->begin
                    tmp += (i_d1 < size(A,d1) ? (@nref $N A d2->(d2==d1)?i_d2+1:i_d2) : (@nref $N A i)
                end
                # Do the shift by -1.
                @nexprs $N d1->begin
                    tmp += (i_d1 > 1 ? (@nref $N A d2->(d2==d1)?i_d2-1:i_d2) : (@nref $N A i)
                end
                # Subtract the center and store the result
                (@nref $N B i) = tmp - 2*$N*(@nref $N A i)
            end
            B
        end
    end
end
```


## Macro reference

### Core macros

```
@nloops N itersym rangeexpr bodyexpr
```
Generate `N` nested loops, using `itersym` as the prefix for the iteration variables. `rangeexpr` may be an anonymous-function expression, or a simple symbol `var` in which case the range is `1:size(var,d)` for dimension `d`.

<br />
```
@nref N A indexexpr
```
Generate expressions like `A[i1,i2,...]`. `indexexpr` can either be an iteration-symbol prefix, or an anonymous-function expression.

<br />
```
@nexpr N expr
```
Generate `N` expressions. `expr` should be an anonymous-function expression. See the `laplacian` example above.

<br />
```
@nextract N esym isym
```
Given a tuple or vector `I` of length `N`, `@nextract 3 I I` would generate the expression `I1, I2, I3 = I`, thereby extracting each element of `I` into a separate variable.

<br />
```
@nall N expr
```
`@nall 3 d->(i_d > 1)` would generate the expression
`(i1 > 1 && i2 > 1 && i3 > 1)`. This can be convenient for bounds-checking.

<br />
```
P, k = @nlinear N A indexexpr
```
Given an `Array` or `SubArray` `A`, `P, k = @nlinear N A indexexpr` generates an
array `P` and a linear index `k` for which `P[k]` is equivalent to
`@nref N A indexexpr`. Use this when it would be most convenient to implement an
algorithm
using linear-indexing. This can be convenient when an anonymous-function
expression cannot not be evaluated at compile-time. For example, using an
example from `Images`, suppose you wanted to iterate over each pixel and perform
a calculation base on the color dimension of an array:

```
sz = [size(img,d) for d = 1:ndims(img)]
cd = colordim(img)  # determine which dimension of img represents color
sz[cd] = 1          # we'll "iterate" over color separately
strd = stride(img, cd)
@nextract $N sz sz
A = data(img)
@nloops $N i d->1:sz_d begin
    P, k = @nlinear $N A i
    rgbval = rgb(P[k], P[k+strd], P[k+2strd])
end
```

It appears to be difficult to generate code like this just using `@nref`,
because the expression `d->(d==cd)` could not be evaluated at compile-time.


### Additional macros

```
@ntuple N itersym
@ntuple N expr
```
Generates an `N`-tuple from a symbol prefix (e.g., `(i1,i2,...)`) or an anonymous-function expression.

```
@nrefshift N A i j
```
Generates `A[i1+j1,i2+j2,...]`. This is legacy from before `@nref` accepted anonymous-function expressions.

```
@nlookup N A I i
```
Generates `A[I1[i1], I2[i2], ...]`. This can also be easily achieved with `@nref`.

```
@indexedvariable N i
```
Generates the expression `iN`, e.g., `@indexedvariable 2 i` would generate `i2`.

```
@forcartesian itersym sz bodyexpr
```
This is the oldest macro in the collection, and quite an outlier in terms of functionality:
```
sz = [5,3]
@forcartesian c sz begin
    println(c)
end
```

This generates the following output:
```
[1, 1]
[2, 1]
[3, 1]
[4, 1]
[5, 1]
[1, 2]
[2, 2]
[3, 2]
[4, 2]
[5, 2]
[1, 3]
[2, 3]
[3, 3]
[4, 3]
[5, 3]
```

From the simple example above, `@forcartesian` generates a block of code like this:

```julia
if !(isempty(sz) || prod(sz) == 0)
    N = length(sz)
    c = ones(Int, N)
    sz1 = sz[1]
    isdone = false
    while !isdone
        println(c)           # This is whatever code we put inside the "loop"
        if (c[1]+=1) > sz1
            idim = 1
            while c[idim] > sz[idim] && idim < N
                c[idim] = 1
                idim += 1
                c[idim] += 1
            end
            isdone = c[end] > sz[end]
        end
    end
end
```

This has more overhead than the direct for-loop approach of `@nloops`, but for many algorithms this shouldn't matter. Its advantage is that the dimensionality does not need to be known at compile-time.


## Performance improvements for SubArrays

Julia currently lacks efficient linear-indexing for generic `SubArrays`.
Consequently, cartesian indexing is currently the only high-performance way to
address elements of `SubArray`s. Many simple algorithms, like `sum`, can have
their performance boosted immensely by implementing them for `SubArray`s using
cartesian indexing. For example, in 3d it's easy to get a boost on the scale of
100-fold this way.
