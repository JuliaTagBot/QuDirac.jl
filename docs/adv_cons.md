# `Dict`-based Construction of States and Operators
---

---
# Functionally Defining Operators
---

Mathematically, one can represent an operator `Ô` with the following definition:

```
Ô | i ⟩ = ∑ⱼ cᵢⱼ | j ⟩
```

In QuDirac, we can define functions that act like `Ô`. These functions are normal 
Julia functions, and can be applied to Kets via the `*` operator. The only 
requirement of these functions is that their argument be a singular `StateLabel`, 
and that they produce a Ket.

Thus, we can define a lowering function like so:

```julia
julia> lower(s::StateLabel) = sqrt(s[1]) * ket(s[1]-1)
lower (generic function with 1 method)

julia> lower * ket(42)
Ket{KroneckerDelta,1,Float64} with 1 state(s):
  6.48074069840786 | 41 ⟩

julia> lower * sum(ket, 1:10)
Ket{KroneckerDelta,1,Float64} with 10 state(s):
  3.1622776601683795 | 9 ⟩
  2.23606797749979 | 4 ⟩
  3.0 | 8 ⟩
  2.0 | 3 ⟩
  2.8284271247461903 | 7 ⟩
  2.6457513110645907 | 6 ⟩
  1.7320508075688772 | 2 ⟩
  2.449489742783178 | 5 ⟩
  1.0 | 0 ⟩
  1.4142135623730951 | 1 ⟩
```

Being able to apply normal Julia functions to Kets is nice, but the syntax for 
doing so can be annoying to use.

Enter QuDirac's `@def_op` macro, which allows one to define an operator-function using
more natural syntax:

```julia
# define the lowering function as "a"
julia> @def_op " a | n > = √n * | n-1 > "
a (generic function with 2 methods)

julia> d" a * | 42 > "
Ket{KroneckerDelta,1,Float64} with 1 state(s):
  6.48074069840786 | 41 ⟩

julia> a * sum(ket, 1:10)
Ket{KroneckerDelta,1,Float64} with 10 state(s):
  3.1622776601683795 | 9 ⟩
  2.23606797749979 | 4 ⟩
  3.0 | 8 ⟩
  2.0 | 3 ⟩
  2.8284271247461903 | 7 ⟩
  2.6457513110645907 | 6 ⟩
  1.7320508075688772 | 2 ⟩
  2.449489742783178 | 5 ⟩
  1.0 | 0 ⟩
  1.4142135623730951 | 1 ⟩

# works with product states
julia> @def_op " a₂ | x,y,z >  = √y * | x,y-1,z >"
a₂ (generic function with 2 methods)

julia> d" a₂ * (| 0,10,8 > - | 12,31,838 >) "
Ket{KroneckerDelta,3,Float64} with 2 state(s):
  3.1622776601683795 | 0,9,8 ⟩
  -5.5677643628300215 | 12,30,838 ⟩
```

The grammar of the string passed to `@def_op` is:

```julia
@def_op "$op_name | $label_args > = f($label_args...) "
```

where `f` is an arbitrary expanded function on the `$label_args` that
returns a Ket. Allowable syntax for the right-hand side of the equation
is exactly the same [syntax allowed by `d"..."`](d_str.md).

For a final example, let's build a function emulating a Hadamard operator:

```julia
julia> @def_op " h | n > = 1/√2 * ( | 0 > - (-1)^n *| 1 > )"
h (generic function with 2 methods)

julia> d" h * | 0 > "
Ket{KroneckerDelta,1,Float64} with 2 state(s):
  0.7071067811865475 | 0 ⟩
  -0.7071067811865475 | 1 ⟩

julia> d" h * | 1 > "
Ket{KroneckerDelta,1,Float64} with 2 state(s):
  0.7071067811865475 | 0 ⟩
  0.7071067811865475 | 1 ⟩
```

---
# Functionally Representing Operators
---

The operator-functions described in the previous example are no doubt useful, 
but they are just normal Julia functions, with the only additional support 
being that they can be applied to Kets using `*`. You can't transpose them, 
or trace over them, etc. They are quite limited when it comes to mimicking 
the behavior of *actual* quantum operators. 

What you *can* do is represent these operators in a basis. This will
yield an `OpSum` object, which can then be treated like an actual operator. 

To generate a representation, we can use the `@repr_op` macro, which acts a lot
like the `@def_op` macro. Here's a familiar example:

```julia
julia> @repr_op " a | n > = √n * | n-1 > " 1:10;

julia> a
OpSum{KroneckerDelta,1,Float64} with 10 operator(s):
  2.449489742783178 | 5 ⟩⟨ 6 |
  3.1622776601683795 | 9 ⟩⟨ 10 |
  2.23606797749979 | 4 ⟩⟨ 5 |
  2.8284271247461903 | 7 ⟩⟨ 8 |
  2.6457513110645907 | 6 ⟩⟨ 7 |
  2.0 | 3 ⟩⟨ 4 |
  1.0 | 0 ⟩⟨ 1 |
  3.0 | 8 ⟩⟨ 9 |
  1.4142135623730951 | 1 ⟩⟨ 2 |
  1.7320508075688772 | 2 ⟩⟨ 3 |
```

The `@repr_op` macro takes in a definition string, and an iterable of items to be used as basis labels.
Notice that the structure of the definition string is *exactly* that of the definition string passed to 
`@def_op`. In fact, the grammar and allowable syntax for the strings passed to each macro is the same. 
The only difference between the two is that the `@repr_op` macro feeds in the given basis labels
to produce an `OpSum`.

Here's an example involving a product basis:

```julia
julia> @repr_op "O | x,y > = normalize( x * | x,y > + y * | y,x > )" [(i,j) for i=1:3, j=1:3]
OpSum{KroneckerDelta,2,Float64} with 15 operator(s):
  0.9486832980505138 | 3,1 ⟩⟨ 3,1 |
  1.0 | 2,2 ⟩⟨ 2,2 |
  1.0 | 3,3 ⟩⟨ 3,3 |
  0.4472135954999579 | 1,2 ⟩⟨ 1,2 |
  0.5547001962252291 | 2,3 ⟩⟨ 2,3 |
  0.8320502943378437 | 3,2 ⟩⟨ 3,2 |
  0.31622776601683794 | 1,3 ⟩⟨ 1,3 |
  0.9486832980505138 | 3,1 ⟩⟨ 1,3 |
  0.8320502943378437 | 3,2 ⟩⟨ 2,3 |
  0.5547001962252291 | 2,3 ⟩⟨ 3,2 |
  ⁞

julia> d" O * | 1, 2 > "
Ket{KroneckerDelta,2,Float64} with 2 state(s):
  0.4472135954999579 | 1,2 ⟩
  0.8944271909999159 | 2,1 ⟩
```

Finally, let's say I already have an operator-function defined, and want
to represent it in a basis. For example, the Hadamard operator-function 
constructed in the previous section:

```julia
julia> @def_op " h | n > = 1/√2 * ( | 0 > - (-1)^n *| 1 > )"
h (generic function with 2 methods)
```

I can easily generate a representation for this function by using `@repr_op` and 
calling `h` on the right-hand side:

```
julia> @repr_op " H | n > = h * | n > " 0:1
OpSum{KroneckerDelta,1,Float64} with 4 operator(s):
  -0.7071067811865475 | 1 ⟩⟨ 0 |
  0.7071067811865475 | 0 ⟩⟨ 0 |
  0.7071067811865475 | 0 ⟩⟨ 1 |
  0.7071067811865475 | 1 ⟩⟨ 1 |
```

The above strategy works to represent any operator-function. Just be aware that
the function and the actual representation need to have unique names:

```
julia> @repr_op " h | n > = h * | n > " 0:1
ERROR: invalid redefinition of constant h
 in anonymous at no file:70
```