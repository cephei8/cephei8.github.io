+++
title = "My Large Array of Things in Odin"
date = 2026-04-05
tags = ["Odin"]
draft = false
+++

## Background {#background}

If you are not familiar with the concept of Large Array of Things, I highly recommend watching [Avoiding Modern C++ | Anton Mikhailov](https://youtu.be/ShSGHb65f3M?si=yp0go2e1-pHUlJCY) on Wookash Podcast, both for educational purposes and for inspiration.<br />
In two words - it's an array of fat structs, with intrusive data structures to implement hierarchies.

In this post I will show my implementation of the concept in Odin, demonstrating some nice features that Odin has to offer.
Particularly:

-   Structure of Arrays (SoA)
-   Intrusive list
-   Fixed size dynamic array


## Design {#design}

There were certain design decisions made that influenced the implementation:

-   There must be no dynamic allocations
-   Handles to the array elements must be stable, in the sense that it is possible to tell if the handle is still valid
-   I wanted to use standard library as much as possible
-   I wanted to explore an idea of using pointer-based handles instead of the index-based handles


### Index-based vs pointer-based handles {#index-based-vs-pointer-based-handles}

The Large Array of Things, as presented by Anton on the Wookash podcast, relied on using indexes of elements in the array.<br />
For example, `first_child_idx` and `next_sibling_idx` are both indexes to other elements in the array.

In my case, I wanted to use Odin's `core:container/intrusive/list` instead of implementing my own intrusive list.<br />
Since using it will give me pointers to elements (e.g. when iterating), I decided to explore the idea of using pointer-based handles in general.<br />
It's not without tradeoffs though - I'll describe them further in the post.


## Implementation {#implementation}


### Entity and its handle {#entity-and-its-handle}

Entity (our fat struct) must contain generation (`_gen`) to be able to uniquely identify itself - this is needed for the cases where the entity is reused (still same pointer/index, but now it's a different entity).<br />
Additionally, **generation is used to determine if the entity is in use: odd generation means the entity is in use, even generation means it is free and ready for reuse**.

The hierarchical relationship is build on top of Odin's `core:container/intrusive/list`.

```odin
import "core:container/intrusive/list"

Entity_Kind :: enum {
	Nil,
	Inventory,
	Item,
}

Entity :: struct {
	_gen:       Entity_Gen,

	kind:       Entity_Kind,
	pos:        [2]u32,

    group:      list.List,
	group_node: list.Node,
}
```

Entity handle (`Entity_Ref`) is simply a pointer + generation (generation at the moment when the handle was created).<br />
Checking if the handle is still valid is as simple as comparing the generation from the pointer (might have been already updated) to the memoized generation.

```odin
Entity_Gen :: u32

Entity_Ref :: struct {
	ptr:  Entity_Ptr,
	_gen: Entity_Gen,
}

entity_ref_valid :: proc(r: Entity_Ref) -> bool {
	return r.ptr._gen == r._gen && r._gen % 2 == 1
}
```

Similarly, generation is used to check if the entity should be skipped when iteration over the pool entities:

```odin
entity_skip :: proc(e: Entity_Ptr) -> bool {
	return e._gen % 2 == 0
}
```


### Entity pool {#entity-pool}

Entity pool is implemented as a SoA array of entities and an array of free entities:

```odin
Entity_Pool :: struct {
	entities:       #soa[MAX_NUM_ENTITIES]Entity,
	upper_bound:    int,
	_free_entities: [dynamic; MAX_NUM_ENTITIES]Entity_Ptr,
}
```

To add to the pool, first a free entity is attempted to be reused, and if not possible - the pool of entities is "extended":

```odin
entity_pool_add :: proc(p: ^Entity_Pool, v: Entity) -> Entity_Ref {
	e: Entity_Ptr
	if len(p._free_entities) > 0 {
		e = pop(&p._free_entities)
	} else {
		assert(p.upper_bound < MAX_NUM_ENTITIES)
		e = &p.entities[p.upper_bound]
		p.upper_bound += 1
	}

	assert(e._gen % 2 == 0)
	gen := e._gen + 1
	e^ = v
	e._gen = gen

	return Entity_Ref{e, e._gen}
}
```

When removing from the pool, the removed entity is added to the array of free entities:

```odin
entity_pool_remove :: proc(p: ^Entity_Pool, r: Entity_Ref) {
	if !entity_ref_valid(r) {
		return
	}

	assert(r.ptr._gen % 2 == 1)
	gen := r.ptr._gen + 1
	r.ptr^ = {}
	r.ptr._gen = gen

	append(&p._free_entities, r.ptr)
}
```

Iterating over the pool entities needs to take free entities into account (`entity_skip`):

```odin
for i in 0 ..< pool.upper_bound {
    e := &pool.entities[i]
    if entity_skip(e) {
        continue
	}

    // do something with the entity
}
```

Here's how working with the hierarchical relationship looks like:

```odin
// import "core:container/intrusive/list"

inventory := Entity{kind = .Inventory}
item1 := Entity{kind = .Item}
item2 := Entity{kind = .Item}

list.push_back(&inventory.group, &item1.group_node)
list.push_back(&inventory.group, &item2.group_node)

it := list.iterator_head(inventory.group, Entity, "group_node")
for e in list.iterate_next(&it) {
    // do something with the entity
}
```


#### Tradeoff of using Odin's intrusive list {#tradeoff-of-using-odin-s-intrusive-list}

Iterating over the list gives `Entity`, not `Entity_Ref` - so at this point it's impossible to check if the entity is valid (it's possible to check if the entity is still used, but impossible to check if it's still from the same generation or it was already recreated).<br />
Code that modifies entities is required to properly handle the hierarchical relationship (e.g. remove the entity from the intrusive list before invalidating it).

If the intrusive list was implemented using indexes - it'd be possible to have both `first_child_idx` and `next_sibling_idx` to be a pair of index + generation, and that would allow to check if the entity generation is still the same.


### Entity pool without SoA {#entity-pool-without-soa}

As of now, fixed size dynamic arrays don't support `#soa`, so a dedicated `upper_bound` is needed.<br />
Howevew, if you don't need SoA, you could just make `entities` to be fixed size dynamic array:

```odin
Entity_Pool :: struct {
	entities:       [dynamic; MAX_NUM_ENTITIES]Entity,
	_free_entities: [dynamic; MAX_NUM_ENTITIES]Entity_Ptr,
}
```


### Serialization and deserialization {#serialization-and-deserialization}

Although much more trivial with index-based handles, serialization and deserialization is still manageable with pointer-based handles.<br />
It will be required to calculate the pointer offset and use entity indexes when serializing/deserializing.


### Implementation gaps {#implementation-gaps}

The demostrated implementation has some gaps to fill before using it for anything serious.<br />
E.g. `Entity_Gen` may overflow if entity is recreated many times - `entity_pool_add` should properly handle this.


## Conclusion {#conclusion}

In this post, implementation of Large Array of Things in Odin was showcased.<br />
The implementation uses Odin's `#soa` and `core:container/intrusive/list`, and demonstrates usage of pointer-based handles.
