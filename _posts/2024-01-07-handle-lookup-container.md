---
layout: post
title: "Handle Lookup Container (Sparse Set)"
categories: c++ lookup handle
---

## Topic

An overview of the handle lookup container (often referred to as a _sparse set_) and an analysis of one implementation.

## Motivation

It is frequently useful to refer to objects indirectly via an id or handle while also ensuring that data is stored contiguously in memory for fast iteration/traversal. By trading off an increase in memory we can get more safety and better performance.

## Example

The core concept behind an id lookup table or sparse set is to add an extra layer of indirection to our data structure [^1]. If we imagine a normal array, instead of indexing elements directly based on their position in the array, we have an id or handle that we use from a parallel array to map to the final position in our backing store.

The great thing about this is our handle can remain stable, but behind the scenes we can be reordering and swapping elements in the backing store while still knowing exactly where our element is (based on the book keeping being done by the container). The advantage to this is we can add and remove elements without worrying about creating 'holes' in the backing store (which interfere with optimum cache utilization). Lookups are bit more expensive (though not hugely) and traversal is as fast as in a traditional array.

The other advantage to these containers is the notion of 'generation'. When we ask for an object to be created we get a handle returned to us. The handle will contain an index to a position in the id lookup part of the table and a generation for the handle. If somewhere else in the program the same element is removed/deallocated, the lookup value will be cleared for that internal handle so we know that object no longer exists. However, we can reuse that same handle again when allocating a new object. When doing so we ensure to increment the generation of the handle. The next time we attempt to resolve the object from our handle, we first check the lookup value is valid, and second check the generations are equal. If the generation of our handle doesn't match the generation of the handle in the same position in the container, we know the handle has been reused and our object no longer exists. This is essentially the same behavior as a weak pointer and is incredibly helpful to avoid lifetime pitfalls (e.g. dangling pointers and use after free).

For more information on this technique please refer to [Managing Decoupling Part 4 -- The ID Lookup Table](http://bitsquid.blogspot.com/2011/09/managing-decoupling-part-4-id-lookup.html) by Niklas Gray. It has a great summary of this approach and a few reasons why to prefer it over other data structures or smart pointers.

For the rest of this post we'll look at a few implementation details of [cpp-handle-container](https://github.com/pr0g/cpp-handle-container). This is an implementation of the above data structure (based on the article by Niklas Gray with a few updates).

[^1]: _"All problems in computer science can be solved by another level of indirection"_ - Butler Lampson

### Resolve vs Call

In order to allocate an element from the container we must first call `add`. This will reserve storage for an element behind the scenes and return a handle to us. When we need to access the object, we call `resolve(handle)`.

```c++
handle_vector<Entity> entities;
handle entity_handle = entities.add();
Entity* entity = entities.resolve(entity_handle);
```

One unfortunate sharp edge to this design is a problem that can occur if we call `resolve` (caching a pointer to our object) and then decide to call `add` a few more times. As the container is backed by a `std::vector`, when `size` outgrows `capacity` the container will get resized and all iterators/pointers will be invalidated. This means if we then try and use our initially resolved pointer, we'll be dealing with a dangling pointer and could get a use after free error or any kind of undefined behavior.

One idea to resolve this problem is to make doing this much more difficult by introducing a member function named `call`.

```c++
handle_vector<Entity> entities;
handle entity_handle = entities.add();
entities.call(entity_handle, [](Entity& entity) {
    // do something with entity...
});
```

This new API works in much the same way `std::for_each` does. The initial parameter is the handle (iterator pair in the case of `for_each`) and the second is the resolved object. If the handle can't be resolved then we simply don't call the function (this does neatly cut out the boilerplate `if (entity)` we'd need if we were using the resolve version, and we can call `has(entity_handle)` to check for validity if we wish).

Now of course if you wanted to you could still capture a pointer from the outside scope and have it point to `Entity& entity`, but you have to go out of your way to do this and it makes doing the wrong thing a bit more difficult. The downside is now maybe the calling syntax is a little uglier and slightly more unwieldy, but that's the thing with computer science and software engineering, "_There are no solutions, there are only trade-offs_" - Thomas Sowell.

How would you solve this problem? What tradeoffs would you make?

### Strong Typing

One other downside to using handles is we can get into situations where it's difficult to know which handle refers to which container. Suppose we have two handle containers (possibly with different types) and we accidentally try to resolve a handle from container A with container B. The compiler isn't going to warn us about this. Hopefully the handle will fail to resolve but then again it might just happen to work and we've accessed an element we shouldn't have which is most likely going to be a horrible bug to track down.

One potential solution to this problem is to (optionally) add strong typing to each container. We can do this with the fantastic 'Phantom Types' trick (see [this article](https://blog.demofox.org/2015/02/05/getting-strongly-typed-typedefs-using-phantom-types/) for a great introduction to the topic).

```c++
struct default_tag{};

using handle = typed_handle<default_tag>;

template<typename T, typename tag = default_tag>
class handle_vector
{
    ...

    typed_handle<tag> add();

    ...
};
```

When we create a container we can provide an unused `tag` template argument to ensure this container is unique. We then use that tag in the `typed_handle` class returned by `add` which makes misusing handles between containers much more difficult.

This probably isn't something to use everywhere, and there's a sensible default to use if strong typing isn't required. Whether this flexibility is worth the cost will depend on the situation. This approach also likely has the added cost of increased compile times (though you'd need to measure to be sure).

### Generations

The last issue to address (or ignore as the case may be) is the question of generation depletion. The generation stored in a handle may eventually grow too large to fit in the type specified for the generation (this can happen if we're frequently reusing the same handles).

In [cpp-handle-container](https://github.com/pr0g/cpp-handle-container), both `Index` and `Generation` are templated, so different sizes can be selected at compile time based on the situation. We could just pick `int64_t` and be done with it, but we may then end up wasting a lot of space that we never need to use. Internally the container keeps track of the generation of the handle, and when it exceeds the value it can potentially store, that slot is marked as 'depleted'. This slot effectively becomes unusable because a situation could feasibly arise where an existing handle can resolve a different object because the generation has wrapped back around to 0 (or another existing generation).

Depleting a handle is undesirable but it prevents an insidious bug (accessing the wrong object unknowingly) which for my money is preferable. As a user of the library it's advisable to periodically track how many depleted handles there are and bump the generation size if this is detected frequently. This is just one possible design and one that in practice might be overkill, what alternative approach would you have opted for?

## Deliberation

Hopefully this post has piqued your curiosity when it comes to alternative/hybrid data structures and the various approaches you can take when implementing them. What are some other containers you'd recommend? How might you implement your own sparse set? I'd be very interested to know! 🙂

## Further Reading

* [Managing Decoupling Part 4 -- The ID Lookup Table](http://bitsquid.blogspot.com/2011/09/managing-decoupling-part-4-id-lookup.html) - Niklas Gray ([GitHub](https://github.com/niklas-ourmachinery))
* [Managing Data Relationships](https://gamesfromwithin.com/managing-data-relationships) - Noel Llopis ([Twitter](https://twitter.com/noel_llopis), [GitHub](https://github.com/llopis))
* [Handles are the better pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html) - Andre Weissflog (FlohOfWoe, Floooh) ([Twitter](https://twitter.com/FlohOfWoe), [GitHub](https://github.com/floooh))
