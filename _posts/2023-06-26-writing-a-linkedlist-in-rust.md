---
layout: post
title: Writing a LinkedList in Rust
author: connor
tags: rust data-structures
---

While Rust already has a [`LinkedList`](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) data structure in the standard library, making your own is a fun and interesting way to learn more about Rust and its features.

By programming your own `linkedlist` in Rust, you'll learn more about Rust's language features, some foundational programming topics and how to work more peacefully alongside the Rust *borrow checker*.

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

These useful properties of the `linkedlist` allow it to use used to model queues of objects or data items in programming problems. You'll be happy you dove down this rabbit hole!

## The Approach

To approach this problem, we will need two data structures:
- A `Node<T>` to store a single data item of generic type `T`
- A `LinkedList<T>` to store a sequence of data items of generic type `T`

Implementing these operations for our `LinkedList<T>` will provide some of the basic functionality we need:
- Create a new empty list
- Push data items to the front or back of the list
- Pop data items from the front or back of the list
- Calculate the length of the list

To balance the potential complexity of our linked-list, we won't be able to:
- Modify data items after they've been added to the list
- Add data items between nodes

These limitations are great opportunities for you to take what we are implemented and extend it further yourselves.

## Storing the data: Node<T>

+++

## Linking the nodes together: LinkedList<T>

+++

## Testing

+++

## Extending further

+++

## Bonus: Iterating through the LinkedList

+++
