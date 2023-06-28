---
layout: post
title: Writing a LinkedList in Rust
author: connor
tags: rust data-structures deep-dive
---

While Rust already has a [`LinkedList`](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) data structure in the standard library, making your own is a fun and interesting way to learn more about Rust.

By programming your own `LinkedList` in Rust, you'll learn more about Rust's language features, some foundational programming topics and how to work more peacefully alongside the infamous Rust *borrow checker*.

Let's get started!

- Generics
- Smart pointers
- Interior mutability
- Borrow checker
- Iterators

## The Problem

For a quick recap, a `linkedlist` allows for data to be stored in nodes that point to other nodes in sequence. A doubly `linkedlist` has nodes that point to previous node and a next node.

`list`-type data structures are typically stored in contiguous regions of memory that are resized to accomodate more elements as they grow in size. The reallocation and copying of `list` elements takes an increasing amount of time, growing in proportion to the size of the `list`. While adding elements to a `list` data structure runs with amortised constant time complexity, the periodic reallocation and copying may present as noticeable performance impacts at large sizes.

`linkedlist`-type data structures are typically stored in dynamically-allocated heap memory, with the nodes not necessarily stored in a particular order. Adding and removing items from a `linkedlist` will always be a constant time operation without the additional overhead of reallocating and copying its elements into larger sections of memory.

These useful properties of the `linkedlist` allow it to use used to model queues and stacks of objects or data items in programming problems. You'll be happy you dove down this rabbit hole!

## The Approach

To approach this problem in Rust, we will need two data structures:
- A `Node<T>` to store a single data item of generic type `T`
- A `LinkedList<T>` to store a sequence of data items of generic type `T`

Implementing these operations for our `LinkedList<T>` will provide some of the basic functionality we need:
- Create a new empty list
- Push data items to the front or back of the list
- Pop data items from the front or back of the list
- Calculate the length of the list

To balance the potential complexity of our `LinkedList<T>`, our implementation won't cover these operations:
- Modify data items after they've been added to the list
- Add data items between nodes

These limitations are great opportunities for you to take what we are implemented and extend it further yourselves.

## Storing the data: Node

Let's start by defining our `Node` struct to hold a generic data type `T`. Our `Node` will contain a data item, along with a previous and next `Node` which may or may not be present.

```rs
/// A node containing a data item and links to its previous and next nodes.
struct Node<T> {
    data: Rc<T>,
    prev: Link<T>,
    next: Link<T>,
}
```

By storing our data item in a referenced-countered pointer (`Rc`), there is no need to clone or copy the data when creating a new node.

### Link

Rust needs to know the size of structs at compile-time, which causes some issues for self-referrencing data type. If we used, we get a compiler error message like the below:

```
error[E0072]: recursive type `Node` has infinite size
   --> src/data_structures/linkedlist.rs:127:1
    |
127 | struct Node<T> {
    | ^^^^^^^^^^^^^^
128 |     data: Rc<T>,
129 |     prev: Node<T>,
    |           ------- recursive without indirection
```

The compiler error message is giving us a hint for how to proceed. We are being nudged towards using *indirection*, which is another way to talk about pointers.

To deal with this compiler error, we need to provide some kind of pointer in place of direct copy of a `Node`. We can define a custom type `Link<T>` as an *indirect* wrapper for our `Node`:

```rs
type Link<T> = Option<Rc<RefCell<Box<Node<T>>>>>;
```

There is a lot packed into this single expression, so let's break down the parts from inside-out:
- `Node<T>`: This is the `Node` that the `Link` connects the current `Node` to
- `Box<T>`: Using boxes enables us to store the `Node<T>` on the heap and store a pointer. The pointer is a constant size (which depends on your computer's architecture, e.g., 64-bit) and helps us to deal with the self-referential type issue.
- `RefCell<T>`: The `RefCell<T>` type allows us to leverage the *interior mutability* design pattern provided by Rust. Since our `Node<T>` is stored on the heap inside of a `Box`, we need to the Rust borrowing rules to be enforced at runtime rather than compile time. By taking advantage of iterior mutability, we can engage more flexibly with the Rust borrowing rules.
- `Rc<T>`: In many cases with Rust, it is clear that each variable has a single owner. However, each `Node` will have multiple owners, including its previous and next node, and the overarching `LinkedList` itself for the head and tail nodes. In our case, we need to have a *reference-counted* pointer to keep track of our multiple owners and allow the Rust compiler to determine when to drop the variable when it no longer has any owners.
- `Option<T>`: Since our `Node` may or may not have another `Node` before or after it, we need to represent this duality. In Rust, we achieve this *optional* state with the `Option<T>` enum. If there is no link from a `Node`, the `prev` or `next` field for our `Node` will be the `None` variant of `Option<T>`. Otherwise, the fields will be an instance of the `Some()` variant containing a reference-counted pointer `Rc<T>`.

As a convience for our `LinkedList` implemention to come, we will define a static method to create a new `Link<T>` from a data item `T`:

```rs
impl<T> Node<T> {
    /// Creates a new Link containing the given data item.
    fn new_link(data: T) -> Link<T> {
        Some(Rc::new(RefCell::new(Box::new(Node::new(data)))))
    }
}
```

### Methods

+++

## Linking the nodes together: LinkedList

+++

## Testing

+++

## Extending further

+++

## Bonus: Iterating through the LinkedList

+++
