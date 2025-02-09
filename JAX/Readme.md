JAX for Python crash course
================
Darren Wilkinson

JAX is a pure functional language embedded in python, but designed to
feel as much like python as practical. From a `python` prompt, first do
some imports.

``` python
import os
import pandas as pd
import numpy as np
import scipy as sp

import jax
from jax import grad, jit
import jax.numpy as jnp
import jax.scipy as jsp
import jax.lax as jl
```

If any of these imports fail, you probably don’t have JAX installed
correctly (in your current environment). For most numpy and scipy
functions, there is a JAX equivalent, so with the above imports,
translating from regular python to JAX often involves replacing a call
to `np.X` (for some `X`) with a call to `jnp.X`, and `sp.X` with
`jsp.X`. But there are other issues to confront, due to the fact that
JAX is a pure functional language and python most definitely isn’t!

## Immutable collections

``` python
v = jnp.array([2, 4, 6, 3]).astype(jnp.float32)
```

    No GPU/TPU found, falling back to CPU. (Set TF_CPP_MIN_LOG_LEVEL=0 and rerun for more info.)

Note that the type of the array has been set to `float32`, since these
are fast and efficient, especially on GPUs. JAX arrays are immutable.

``` python
v[2]
vu = v.at[2].set(7)
vu
```

    Array([2., 4., 7., 3.], dtype=float32)

``` python
v
```

    Array([2., 4., 6., 3.], dtype=float32)

We can map JAX arrays.

``` python
jl.map(lambda x: 2*x, v)
```

    Array([ 4.,  8., 12.,  6.], dtype=float32)

We can also reduce them.

``` python
jl.reduce(v, 0.0, lambda x,y: x+y, [0])
```

    Array(15., dtype=float32)

``` python
jnp.sum(v)
```

    Array(15., dtype=float32)

The reduction must be *monoidal* (the operation must be associative, and
the initial value must be an identity wrt that operation), or the result
is undefined.

## Functions

Functions are written like regular python functions. But if they are to
be part of a hot loop, they can be JIT-compiled.

``` python
@jit
def sumArray1d(v):
  return jl.reduce(v, 0.0, lambda x,y: x+y, [0])

float(sumArray1d(v))
```

    15.0

Note that you can’t use `float` inside a JIT’d JAX function, since
`float` is a python function, not a JAX function.

We have seen that functional languages often exploit recursion, either
explicitly or implicitly, for the implementation of “looping”
constructs. However, allowing general recursion turns out to be
problematic for reverse-mode automatic differentiation. Consequently,
some differentiable functional languages (such as JAX and Dex) disallow
recursive functions. But without any mutable variables or recursion, how
can we loop?! In this case the language must provide us with some
built-in constructs. In JAX, the two most commonly used constructs (in
addition to
[`map`](https://jax.readthedocs.io/en/latest/_autosummary/jax.lax.map.html),
[`reduce`](https://jax.readthedocs.io/en/latest/_autosummary/jax.lax.reduce.html)
and
[`scan`](https://jax.readthedocs.io/en/latest/_autosummary/jax.lax.scan.html))
are
[`jax.lax.while_loop`](https://jax.readthedocs.io/en/latest/_autosummary/jax.lax.while_loop.html)
and
[`jax.lax.fori_loop`](https://jax.readthedocs.io/en/latest/_autosummary/jax.lax.fori_loop.html).
Note that you cannot reverse-mode differentiate through a `while_loop`
(this is problematic for the same reason that recursive functions are
problematic - you cannot know statically what the memory requirements
will be). A for loop is relatively straightforward.

``` python
def logFactF(n):
  return float(jl.fori_loop(1, n+1,
    lambda i,acc: acc + jnp.log(i), 0.0))

logFactF(3)
```

    1.7917594909667969

``` python
logFactF(100000)
```

    1051299.625

Note that the upper bound on the loop is *exclusive*. A while loop is
slightly more involved, due to the need to propagate two items of state
(the counter and the accumulator). However, the while loop can be used
when the number of iterations is not known statically.

``` python
def logFactW(n):
  def cont(state):
    [i, acc] = state
    return i <= n
  def advance(state):
    [i, acc] = state
    return [i + 1, acc + jnp.log(i)]
  return float(jl.while_loop(cont, advance, [1, 0.0])[1])

logFactW(3)
```

    1.7917594909667969

``` python
logFactW(100000)
```

    1051299.625
