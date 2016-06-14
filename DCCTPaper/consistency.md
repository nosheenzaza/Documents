# Declarative consistency policies

Following [Quelea], we define consistency policies using first-order logic.

In the model, a client performs _operations_ on objects in a replicated data store.
Each operation is executed atomically at a replica, but not necessarily at all
replicas. Operations `o` access exactly one object `object(o)`.

Operations consist of more primitive read and write operations on registers with LWW semantics. In the degenerate case, objects are registers and an operation is just a read or a write.
Operations may be read-only or read-write.

Objects are stamped with a unique version in the model (but not necessarily in the implementation).
There is no ordering on versions; they are just unique identifiers. We can get rid of versions by just comparing values in the definitions below. This might introduce ABA problems, however.

Objects `x` are grouped into _regions_ `r`. We say `region(x) = r`.
In the degenerate case, each variable is its own region (`region(x) = x`).
Regions do not overlap, although this could be relaxed.

(Trans)actions are sequences of one or more operations.
We introduce the primitive operations `begin(a)`, `commit(a)`, and `abort(a)`, where `a` is an action.
For each action `a`, `begin(a)` must occur before `commit(a)` or `abort(a)` in session order (defined below).
There is exactly one `begin(a)` operation and exactly one `commit` or `abort`.
An action may perform operations on more than one region.
A client performs actions sequentially. We can model a concurrent client as multiple clients.

## Predicates

Define the following predicates:

**Session order**: If operation `o` is executed before operation `p` at the same client
(i.e., in the same session), `so(o,p)` holds.
`so(o,p)` and `so(p,q)` implies `so(o,q)`.

**Actions**: An operation `o` belongs to action `a` if `so(begin(a), o)` and `so(o, commit(a))` or `so(o, abort(a))`.
We write `action(o,a)`. We can lift `so` to actions: `so(a,b)` if `so(begin(a), begin(b))`.
Actions are executed sequentially on the client: if `so(begin(a), begin(b))`, then `so(commit(t), begin(u))` or `so(abort(t), begin(u))`.
We say `commits(t)` if there is a `commit(t)` operation and `aborts(t)` if there is an `abort(t)` operation.
We say `commits(a)` if `commits(t)` and `action(a,t)`.

**Region accesses**:
`reads(o,r)` if operation `o` reads from region `r`.
`writes(o,r)` if operation `o` writes to region `r`.
`accesses(o,r)` if `reads(o,r)` or `writes(o,r)`.
We can lift these definitions to actions: `accesses(a,r)` if there is an operation `o` where `action(o,a)` and `accesses(o,r)`.
Similarly for `reads` and `writes`.

**Session order for regions**:
`sor(a,b,r)` = `so(a,b)` and `accesses(a,r)` and `accesses(b,r)`.
We can lift this definition to actions. **TODO: not needed**.

**Output dependency**: If operation `o` writes version `v` of object `x` and operation `p` overwrites version `v` of `x`, then `ww(o, p)`.
We can lift this definition to actions: `ww(a,b,r)` if there is some `o` with `action(o,a)` and some `p` with `action(p,b)`, where
`region(a) = region(b) = r` and `ww(o,p,r)`. **TODO: where? at what replica? one or all?**

**Flow dependency**: If operation `o` writes version `v` of object `x` and operation `p` reads version `v` of `x`, then `wr(o, p)`.
We can lift this definition to actions: `wr(a,b,r)`.

**Anti dependency**: If operation `o` reads version `v` of object `x` and operation `p` overwrites version `v` of `x`, then `rw(o, p)`.
We can lift this definition to actions: `rw(a,b,r)`.

**Visibility**: `vis(o,p,r)` is the transitive closure of the union of `ww(o,p,r)` and `wr(o,p,r)`.
We can lift this definition to actions: `vis(a,b,r)`.  **TODO: this definition might be incompatible with [Quelea].**

**Dependency**: `depends(o,p,r)` is the transitive closure of the union of `ww(o,p,r)`, `wr(o,p,r)`, and `rw(o,p,r)`.
We can lift this definition to actions: `depends(a,b,r)`.

**Happens before**: 
`hbr(a,b,r)` is the transitive closure of the union of `sor(a,b,r)` and `vis(a,b,r)`.
`hb(a,b)` is the transitive closure of the union of `so(a,b)` and `vis(a,b)`.

## Consistency policies

Each region has a _consistency policy_ `policy(r)`.

A region `r` satisfies **serializability** if `depends(*,*,r)` is acyclic (where actions are considered nodes in a graph and there is an edge `a` to `b` if `depends(a,b,r)`).
**TODO: _no dirty reads_: if `wr(a,b,r)`, then the write to `r` that induced the dependency is the last write to `r` before `commit(a)`.**
**TODO: _no aborted reads_: if `wr(a,b,r)`, then `commits(a)`.**  See [Adya].

**TODO causal consistency**

**TODO eventual consistency**

### Session guarantees

A region `r` satisfies **read your writes** if
for any two actions `a` and `b`, if
`sor(a,b,r)` and `writes(a,r)` and `reads(b,r)`, then
`vis(a,b,r)`.

A region `r` satisfies **monotonic reads** if
for any three actions `a`, `b`, and `c`,
if `writes(a,r)` and `reads(b,r)` and
`sor(a,c,r)` and `vis(a,b,r)`, then `vis(a,c,r)`.

A region `r` satisfies **monotonic writes** if
for any three actions `a`, `b`, and `c`, if
`writes(a,r)` and `writes(b,r)` and `sor(a,b,r)`
and `vis(b,c,r)`, then `vis(a,c,r)`.

A region `r` satisfies **writes follow reads** if
for any four actions `a`, `b`, `c`, and `d`, if
`reads(a,r)`, `writes(b,r)`, and `writes(c,r)`, and
if `vis(a,b,r)` and `vis(c,d,r)` and (`sor(b,c,r)` or `b = c`), then `vis(a,d,r)`.

### Transactional policies

A region `r` satisfies **read committed** if transitive closure of `ww(*,*,r)` union `wr(*,*,r)` on actions is acyclic.

A region `r` satisfies **read uncommitted** if the transitive closure of `ww(*,*,r)` on actions is acyclic.

A region `r` satisfies **snapshot isolation** if .. **TODO: See [Adya].**

A region `r` satisfies **repeatable read** if `action(oa,a)` and `action(pa,a)` and `action(ob,b)` and `action(pb,b)`,
and `vis(ob,oa,r)` and `accesses(pb,r)` and `accesses(pa,r)`, then `vis(pb,pa,r)`.   **This definition is adapted from [Quelea]. Should re-write from [Adya].**

A region `r` satisfies **monotonic atomic views** if `action(o) = action(p)` and `action(o') = action(p')` and `sor(o,p,r)` and `vis(o',o,r)` and `region(p') = region(p) = r`, then `vis(p',p)`.

## References

[Quelea]: Sivaramakrishnan, Kaki, Jagannathan, Declarative Programming over Eventually Consistent Data Stores, _PLDI'15_.

[Adya]: Atul Adya, PhD thesis.
