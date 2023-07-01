---
layout: post
title: Writing a LinkedList in Rust
author: connor
tags: rust data-structures deep-dive
---
<!-- Ensure links open in a new tab when clicked -->
<base target="_blank">

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
/// A node containing a data item and links to previous and next nodes.
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
- [`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html): Using boxes enables us to store the `Node<T>` on the heap and store a pointer. The pointer is a constant size (which depends on your computer's architecture, e.g., 64-bit) and helps us to deal with the self-referential type issue.
- [`RefCell<T>`](https://doc.rust-lang.org/std/cell/struct.RefCell.html): The `RefCell<T>` type allows us to leverage the *interior mutability* design pattern provided by Rust. Since our `Node<T>` is stored on the heap inside of a `Box`, we need to the Rust borrowing rules to be enforced at runtime rather than compile time. By taking advantage of iterior mutability, we can engage more flexibly with the Rust borrowing rules.
- [`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html): In many cases with Rust, it is clear that each variable has a single owner. However, each `Node` will have multiple owners, including its previous and next node, and the overarching `LinkedList` itself for the head and tail nodes. In our case, we need to have a *reference-counted* pointer to keep track of our multiple owners and allow the Rust compiler to determine when to drop the variable when it no longer has any owners.
- [`Option<T>`](https://doc.rust-lang.org/std/option/enum.Option.html): Since our `Node` may or may not have another `Node` before or after it, we need to represent this duality. In Rust, we achieve this *optional* state with the `Option<T>` enum. If there is no link from a `Node`, the `prev` or `next` field for our `Node` will be the `None` variant of `Option<T>`. Otherwise, the fields will be an instance of the `Some()` variant containing a reference-counted pointer `Rc<T>`.

**Note that because our `Link<T>` type uses `Rc` (referenced-counter pointer) instead of `Arc` (atomic referenced-counted pointer), our implementation of `Node` is not thread-safe.**

### Node constructor

Each of our `Node` methods will be enclosed in a block: `impl<T> Node<T> { ... }`
This allows our methods to take data items of the generic type `T`, making our `Node` quite flexible!

Now that he know how our `Node` will link to others, we can implement the `new()` method. Each `Node` will start out holding as a data item of the generic type `T` and not connected to a previous or next `Node`.

```rs
/// Creates a new Node containing the given data item. The previous and next
/// node links are set to None.
fn new(data: T) -> Node<T> {
    Node {
        data: Rc::new(data),
        prev: None,
        next: None,
    }
}
```

### Node setters

Our methods for updating the previous and next nodes will be fairly straightforward. Using our referenced-counted `Link<T>`, we only need to clone the other link when using our setting methods. The logic to ensure two nodes next to each other are pointing at each other correctly will be handed by our `LinkedList`.

```rs
/// Updates the previous node.
fn set_prev(&mut self, other: &Link<T>) {
    self.prev = other.clone();
}

/// Updates the next node.
fn set_next(&mut self, other: &Link<T>) {
    self.next = other.clone();
}
```

### Node getters

In a similar vein to our `Node` setter methods, our getter methods only need to clone the next or previous links.

```rs
/// Gets the previous link from the Node via cloning.
fn get_prev(&self) -> Link<T> {
    self.prev.clone()
}

/// Gets the next link from the Node via cloning.
fn get_next(&self) -> Link<T> {
    self.next.clone()
}
```

In order to get the data stored by our `Node` without needing to clone or copy the data, we can return a clone of the referenced-counted pointer to it.

```rs
/// Gets the data item contained within the Node via cloning.
fn get_data(&self) -> Rc<T> {
    self.data.clone()
}
```

As a convience for our `LinkedList` implemention to come, we will define a static method to create a new `Link<T>` from a data item `T`:

```rs
/// Creates a new Link containing the given data item.
fn new_link(data: T) -> Link<T> {
    Some(Rc::new(RefCell::new(Box::new(Node::new(data)))))
}
```

With our `Node` struct and its methods implemented, we can move onto our `LinkedList` implementation.

## Linking the nodes together: LinkedList

Our `LinkedList` needs to keep track of its first and last nodes (the *head* and *tail* nodes) and its length.

```rs
/// An implementation of a doubly linked-list. Not thread-safe. Note that the
/// data items contained within nodes cannot be changed after they have been
/// added to the linked-list.
pub struct LinkedList<T> {
    head: Link<T>,
    tail: Link<T>,
    len: usize,
}
```

While we could dynamically calculate the length of our `LinkedList`, it is more efficient to keep track of its length as we push and pop data items.

### LinkedList constructor

Our `LinkedList` methods will be enclosed in a block: `impl<T> LinkedList<T> { ... }`. Like with our `Node` methods, this implementation block will allow our `LinkedList` methods to take in data items of the generic data type `T`, making our `LinkedList` flexible like our `Node`!

A new `LinkedList` starts out empty, with no head or tail node and a length of 0.

```rs
/// Creates an empty LinkedList.
pub fn new() -> LinkedList<T> {
    LinkedList {
        head: None,
        tail: None,
        len: 0,
    }
}
```

### Push methods

Our `LinkedList` push methods are where this project starts to get really fun! By convention, "push" by itself means pushing to the end of the `LinkedList` and "pushing to the front" means exactly that.

To push a new node to the end of the list, we first create another `Node` containing the new data item. If the `LinkedList` is empty, the new `Node` becomes the new head and tail. Otherwise, the new node becomes the new tail. The last part of our `push()` method ensures that the old tail and the new tail point to each other and the `LinkedList` is tracking the new tail.

The `as_ref()` method called on our the `Link` instances allows us to borrow the `Rc` held within the `Option` instance, rather than taking ownership of it. With the `Rc` borrowed, we can dynamically borrow the `Node` contained within the `Link` via the `borrow_mut()` method called on the `RefCell` nested within the `Link`.

```rs
/// Pushes the data item to the end of the LinkedList.
pub fn push(&mut self, data: T) {
    let new_node: Link<T> = Node::new_link(data);
    // Increase the len counter
    self.len += 1;
    // Handle case for empty list
    if self.head.is_none() && self.tail.is_none() {
        self.head = new_node.clone();
        self.tail = new_node;
        return;
    }
    // Update the tail to point at the new node and connect to the old tail
    self.tail.as_ref().unwrap().borrow_mut().set_next(&new_node);
    new_node.as_ref().unwrap().borrow_mut().set_prev(&self.tail);
    self.tail = new_node;
}
```

The great part about our `LinkedList` being symmetrical is that our `push_front()` method ends up being similar to `push()`, with references to head/tail and prev/next being swapped around.

```rs
/// Pushes the data item to the front of the LinkedList.
pub fn push_front(&mut self, data: T) {
    let new_node: Link<T> = Node::new_link(data);
    // Increase the len counter
    self.len += 1;
    // Handle case for empty list
    if self.head.is_none() && self.tail.is_none() {
        self.head = new_node.clone();
        self.tail = new_node;
        return;
    }
    // Update the head to point at the new node and connect to the old head
    self.head.as_ref().unwrap().borrow_mut().set_prev(&new_node);
    new_node.as_ref().unwrap().borrow_mut().set_next(&self.head);
    self.head = new_node;
}
```

### Pop methods

The value we get from popping a data item from our `LinkedList` depends on whether or not it has nodes within it. Like our push methods, "pop" by itself refers to removing a value from the end (tail) of the `LinkedList` and "pop from the front" means what it says.

If our `LinkedList` contains at least one `Node`, our `pop()` method will return the value contained in the removed node wrapped in a `Some` from the `Option`. Otherwise, it will return a `None` value indicating that no `Node` was removed.

```rs
/// Removes the last node from the LinkedList. Returns Some containing the value
/// from the removed node, otherwise None.
pub fn pop(&mut self) -> Option<Rc<T>> {
    // Handle case for empty list
    if self.head.is_none() && self.tail.is_none() {
        return None;
    }
    // Update the tail to be the second-last node and return value contained in removed node
    let old_tail = self.tail.clone();
    self.tail = old_tail.as_ref().unwrap().borrow().get_prev();
    self.tail.as_ref().unwrap().borrow_mut().set_next(&None);
    let old_data = old_tail.as_ref().unwrap().borrow().get_data();
    // Decrease the len counter
    self.len -= 1;
    Some(old_data)
}
```

As with the push methods, the pop methods are symmetrical. For the `pop_front()` method, the references to head/tail and prev/next are swapped around compared to the `pop()` method.

```rs
/// Removes the first node from the LinkedList. Returns Some containing the
/// value from the removed node, otherwise None.
pub fn pop_front(&mut self) -> Option<Rc<T>> {
    // Handle case for empty list
    if self.head.is_none() && self.tail.is_none() {
        return None;
    }
    // Update head to be second node and return value contained in removed node
    let old_head = self.head.clone();
    self.head = old_head.as_ref().unwrap().borrow().get_next();
    self.head.as_ref().unwrap().borrow_mut().set_prev(&None);
    let old_data = old_head.as_ref().unwrap().borrow().get_data();
    // Decrease the len counter
    self.len -= 1;
    Some(old_data)
}
```

### Checking length

Checking the length of our `LinkedList` is very straightforward now, since we have been keeping track the nodes as they are pushed and popped.

```rs
/// Returns the number of items contained in the LinkedList.
pub fn len(&self) -> usize {
    self.len
}
```

For the sake of completeness, we should implement an `is_empty()` method because we have a `len()` method. Our `LinkedList` will be empty if the `len` field is equal to 0.

```rs
/// Checks if the LinkedList is empty.
pub fn is_empty(&self) -> bool {
    self.len == 0
}
```

## Testing

It is good practice to write and use test cases for your code. That way you know if your implementation is behaving as expected and have a head's up if something has broken through changes to your code.

Below are some example test methods we can use to check various parts the functionality of our `LinkedList`. These test methods are by no means exclusive - feel free to write more test cases yourself. You can execute the test methods using the `cargo test` command.

```rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_push_back_length() {
        let mut new_list = LinkedList::<i32>::new();
        let values = (0..10).collect::<Vec<i32>>();
        for i in values {
            new_list.push(i);
        }
        assert_eq!(new_list.len(), 10);
    }

    #[test]
    fn test_push_front_length() {
        let mut new_list = LinkedList::<i32>::new();
        let values = (0..10).collect::<Vec<i32>>();
        for i in values {
            new_list.push_front(i)
        }
        assert_eq!(new_list.len(), 10);
    }
}
```

## Wrapping Up

The full source-code of my `LinkedList` implementation can be [found on my GitHub][my-source-file]. I've included some additional test methods and other features beyond what I've covered in this walkthrough, such as implementing iterator features for our `LinkedList`.

Reach out to me on LinkedIn or GitHub and let me know what you think!

[my-source-file]: https://github.com/cmooneycollett/ads-sandbox/blob/7ec4c3cc70c93ac4dc8d6c6cf0bd256563c5fdc9/src/data_structures/linkedlist.rs