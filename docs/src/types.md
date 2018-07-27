#LightGraphs Types

*LightGraphs.jl* supports both the `AbstractGraph` type and two concrete simple graph types- - `SimpleGraph` for undirected graphs and `SimpleDiGraph` for directed graphs -- that are subtypes of `AbstractGraph`.

## Concrete Types

*LightGraphs.jl* provides two concrete graph types: `SimpleGraph` is an undirected graph, and `SimpleDiGraph` is its directed counterpart. Both of these types can be parameterized to specifying how vertices are identified (by default, `SimpleGraph` and `SimpleDiGraph` use the system default integer type, usually `Int64`).

A graph *G* is described by a set of vertices *V* and edges *E*: *G = {V, E}*. *V* is an integer range `1:n`; *E* is represented as forward (and, for directed graphs, backward) adjacency lists indexed by vertices. Edges may also be accessed via an iterator that yields `Edge` types containing `(src<:Integer, dst<:Integer)` values. Both vertices and edges may be integers of any type, and the smallest type that fits the data is recommended in order to save memory.

Graphs are created using `SimpleGraph()` or `SimpleDiGraph()`; there are several options (see the tutorials for examples).

Multiple edges between two given vertices are not allowed: an attempt to add an edge that already exists in a graph will not raise an error. This event can be detected using the return value of `add_edge!`.

## AbstractGraph type

To encourage experimentation and development on top of the JuliaGraphs ecosystem, *LightGraphs.jl* defines the `AbstractGraph` type and associated interface,
which is used by libraries like [MetaGraphs.jl](https://github.com/JuliaGraphs/MetaGraphs.jl) (for graphs with associated meta-data) and [SimpleWeightedGraphs.jl](https://github.com/JuliaGraphs/SimpleWeightedGraphs.jl) (for weighted graphs). All types that are a subset of `AbstractGraph` must implement the following functions (most of which are described in more detail in [Accessing Graph Properties](@ref) and [Making and Modifying Graphs](@ref)):

```@index
Order = [:type, :function]
Pages   = ["types.md"]
```

## Example: building a new graph type from scratch

A graph can be represented in an exhaustive manner by its
[adjacency matrix](https://en.wikipedia.org/wiki/Adjacency_matrix).
We are going to build a type wrapping a sparse matrix for this, given that
graphs tend to be sparse. We can also define custom edge types or re-use
`SimpleEdge` which will be enough in our case.

### Accessing properties

We need to define the behavior required in [Accessing Graph Properties](@ref).

```julia
import LightGraphs: is_directed, ne, nv, edgetype, edges, vertices,
                    outneighbors, inneighbors, has_vertex, has_edge

mutable struct MatrixDiGraph{MT<:AbstractMatrix{Bool}} <: LightGraphs.AbstractGraph{Int}
    m::MT
end
```

Now that we've defined the type, we can implement the required behavior.
```julia
is_directed(::MatrixDiGraph) = true
edgetype(::MatrixDiGraph) = LightGraphs.SimpleGraphs.SimpleEdge{Int}
ne(g::MatrixDiGraph) = sum(g.m)
nv(g::MatrixDiGraph) = size(g.m)[1]

vertices(g::MatrixDiGraph) = 1:nv(g)

function edges(g::MatrixDiGraph)
    n = nv(g)
    return (LightGraphs.SimpleGraphs.SimpleEdge(i,j) for i in 1:n for j in 1:n if g.m[i,j])
end
```

Some methods are also required on vertices (nodes) for the graph types:
```julia
outneighbors(g::MatrixDiGraph,node) = [v for v in 1:nv(g) if g.m[node,v]]
inneighbors(g::MatrixDiGraph,node) =  [v for v in 1:nv(g) if g.m[v,node]]
has_vertex(g::MatrixDiGraph,v::Integer) = v <= nv(g) && v > 0
```

Finally, some methods are required on edges:
```julia
has_edge(g::MatrixDiGraph,i,j) = g.m[i,j]
```

### Modifying graphs

Optionally, we can make our graph type mutable by implementing methods
described in [Making and Modifying Graphs](@ref). All these methods
return true if the operation was successful.

```
import LightGraphs: rem_edge!, rem_vertex!, add_edge!, add_vertex!

## vertex modification

function add_vertex!(g::MatrixDiGraph)
    n = nv(g)
    m = zeros(Bool,n+1,n+1)
    m[1:n,1:n] .= g.m
    g.m = m
    return true
end

function rem_vertex!(g::MatrixDiGraph, v)
    n = nv(g)
    (v < 1) || v > n) && return false
    remaining = setdiff(1:n,v)
    g.m = g.m[remaining,remaining]
    return true
end

## edge modification

function add_edge!(g::MatrixDiGraph, e)
    has_edge(g,e) && return false
    n = nv(g)
    (src(e) > n || dst(e) > n) && return false
    g.m[src(e),dst(e)] = true
end

function rem_edge!(g::MatrixDiGraph,e)
    has_edge(g,e) || return false
    n = nv(g)
    (src(e) > n || dst(e) > n) && return false
    g.m[src(e),dst(e)] = false
    return true
end
```

### Define optional methods

LightGraphs defines, exposes and uses other functions with a default
implementation valid for any graph defining the methods above.
One example is `adjacency_matrix`, which will compute the matrix from the edges.
Since our custom type `MatrixDiGraph` is defined on top of this matrix, we can just return it.

```julia
using BenchmarkTools: @btime # used for reliable bencmarks
import LightGraphs: adjacency_matrix

# using the default implementation
@btime adjacency_matrix(bigger_twitter)
# 3.681 ms (5222 allocations: 682.03 KiB)

adjacency_matrix(g::MatrixDiGraph) = g.m
# using our new method
@btime A = adjacency_matrix(bigger_twitter)
# 13.047 ns (0 allocations: 0 bytes)
```

## Full Docs for AbstractGraph Functions

```@autodocs
Modules = [LightGraphs]
Pages   = ["interface.jl"]
Private = false
```
