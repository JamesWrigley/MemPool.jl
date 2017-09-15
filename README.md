# MemPool

Distributed datastore that supports custom serialization, spilling least recently used data to disk and memory-mapping.

## Usage

```julia
addprocs(4)
using MemPool
@everywhere MemPool.max_memsize[] = 10^9 # 1 GB per worker
```

This sets the memory limit on each process to 10^9 bytes (1GB). If this is exceeded, the least recently used data will be written to disk using `movetodisk` described below until the total pool size is below 1 GB. Data thus spilled are written in a directory called `.mempool`. The data can be read back with memory mapping. Overriding `mmwrite` and `mmread` described in the next section is recommended for efficiency.

Data store functions:

- `poolset(x::Any, pid=myid())`: store the object `x` on `pid`. Returns a `DRef` object.
- `poolget(r::DRef)`: gets the data stored at `DRef`. If the data has been moved to disk, `poolget` will read it from disk on the caller side.
- `pooldelete(r::DRef)`: removes data at `r`, including any data on disk.
- `movetodisk(r::DRef)`: moves data to disk and release it from memory. Uses `MemPool.mmwrite` to write to disk. See section below. Returns a `FileRef` which can be passed to `poolget` to read the data.
- `copytodisk(r::DRef)`: copies data to disk keeping the original copy in memory.
- `savetodisk(r::DRef, path)`: saves data to a given file path. Leaves original data in memory, doesn't affect LRU accounting.


## `MemPool.mmwrite`, `MemPool.mmread`

- `mmwrite(s::AbstractSerializer, x::Any)` is called to write data to the wire / file when data needs to be transferred / written to disk. Packages can define how parts of their datastructure can be written in raw format that can be mmapped back later with `mmread`. `mmwrite` must begin with the command `Base.serialize_type{MemPool.MMSer{typeof(x)}` so that Julia's base serializer will dispatch any deserialization to `mmread`.
- `mmread(::Type{T}, io::AbstractSerializer)` is called to deserialize data written with `mmwrite`.

`mmwrite` can currently store Array{String} much more efficiently than Base. It is also extended for fast storage of NullableArrays, PooledArrays, and IndexedTables by JuliaDB.
