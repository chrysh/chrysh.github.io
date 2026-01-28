---
title: Hashset
date: '2024-08-09'
categories:
  - Rust
  - Linux Kernel
  - Kernel Driver
  - Data Structures
tags:
  - Rust
  - Linux Kernel
  - Kernel Driver
  - Data Structures
  - Hashset
  - Core
---

[<img class="penguin" src="/static/img/rusty_penguin_11.jpeg" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Why reinventing HashSet?

This article will focus on adding a Hashset to the
[quarto_rs](https://github.com/chrysh/quarto_rs/tree/add_hashset). As previously
discussed, because we cannot use the Rust stdlib when writing code for the Linux
kernel, we have to implement the HashSet trait and implementation ourselves.

I am aware of the fact that it would have been much easier to replace the HashSet
with a Vector of size 64 and iterate over it. But I wanted to modify the
original code as little as possible.

## What is a HashSet?

A HashSet is a data structure, that allows you to store and retrieve data using
hashes. A hashing algorithm converts your data into a chunk of fixed size called
hash, which is then used for indexing your data in the HashSet. For example,
your image of size 4 KB can be reduced to a 16 Byte hash value, which then
serves as the index for the hashtable the next time you want to find out whether
this element is already in the HashSet.
Compared to other data structures like lists or Arrays, a HashSet generally uses
less memory space, which becomes especially relevant for bigger array sizes than
what we are dealing with here.

For more information on HashSets, check out the [Rust
documentation](https://doc.rust-lang.org/std/collections/struct.HashSet.html).

## HashSet and Hasher

A HashSet can support any `hasher H` in order to assign data to
each bucket, as long as it implements the trait `Hasher`. This trait implements
two functions: `write` to add data bytes to be included in the hash, as well as
the `finish` method, which returns the calculated hash. The `Hash` and `Hasher`
traits are both defined in `core`.

One easy algorithm to implement as a Hasher is DJB2, which we will use as our
default hasher.

```
use core::hash::Hasher;

#[derive(Debug, Clone, Copy)]
pub struct DJB2Hasher {
    hash: u32,
}

impl DJB2Hasher {
    pub fn new() -> DJB2Hasher {
        DJB2Hasher { hash: 5381 }
    }
}

impl Hasher for DJB2Hasher {
    fn finish(&self) -> u64 {
        self.hash as u64
    }

    fn write(&mut self, bytes: &[u8]) {
        for &byte in bytes {
            self.hash = ((self.hash << 5).wrapping_add(self.hash)).wrapping_add(byte as u32);
        }
    }
}
```

## HashSet trait

Our HashSet implementation has `capacity` number of buckets, that can be filled
with `Some(T)` or be empty, marked by `None`. At each moment, there are `size`
number of filled buckets. By default, DJB2Hasher is used. But the user also has
the possibility to use any other Hasher.

```
#[derive(Debug, Clone)]
pub struct HashSet<T, H = DJB2Hasher>
    where T: Clone
{
    buckets: VecExtra<Option<T>>,
    capacity: usize,
    size: usize,
    hasher: H,
}
```

The functions that HashSet implements are based on the stdlib HashSet trait,
many of which allow you to compare two HashSets for overlaps or differences in
content.

```
    // Constructors
    pub fn with_capacity(capacity: usize) -> Self {}
    pub fn with_hasher_and_capacity(hasher: H, capacity: usize) -> Self {}

    // Helper functions
    fn calculate_index(&mut self, item: &T) -> usize {}
    fn resize(&mut self) {}

    // HashSet functions
    pub fn hash(&mut self, item: &T) -> u64 {}
    pub fn len(&mut self) -> usize {}
    pub fn insert(&mut self, item: T) {}
    pub fn contains(&mut self, item: &T) -> bool {}
    pub fn remove(&mut self, item: &T) {}
    pub fn is_empty(&mut self) -> bool {}
    pub fn difference(&mut self, other: &mut HashSet<T, H>) -> VecExtra<T> {}
    pub fn union(&mut self, other: &mut HashSet<T, H>) -> VecExtra<T> {}
    pub fn intersection(&mut self, other: &mut HashSet<T, H>) -> VecExtra<T> {}
    pub fn symmetric_difference(&mut self, other: &mut HashSet<T, H>) -> VecExtra<T> {}
    pub fn drain(&mut self) {}
```

## HashSetIter

Now we want be able to iterate over the elements of the HashSet, for example
for using the `collect` function to transform HashSet from and to a vector
array, or to iterate over and print each element. In order for this to work, we
need to implement the IntoIterator trait for HashSet, which will return the
Iterator HashSetIter. This Iterator records the most recent index `current_pos`
that was returned, so that on each call to `next` the next element in HashSet is
returned. Since we are working with mutable references, we need to specify the
lifetime `a` of the elements in HashSet.

```
pub struct HashSetIter<'a, T>
where T: Clone
{
    hashset: &'a HashSet<T>,
    current_pos: usize,
}

impl<'a, T: Clone> Iterator for HashSetIter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {}
}
```

## `collect()` into HashSet

Now we want to be able to create a HashSet from a vector, as shown in the next
example:

```
let remaining_pieces_vec = vec![ 1 ,2, 4 ];
  let remaining_pieces = remaining_pieces_vec
     .iter()
     .copied()
     .collect::<HashSet<_>>();
```

In Rust, if you want to use `collect()`  with a type, it needs to implement the
IntoIterator trait. For example, our example of a HashSet uses VecExtra as a
container for the elements. Therefore, the VecExtra type must implement
IntoIter.

Furthermore, for each "flavor" or way of accessing a type, you need to implement
the trait, and therefore the function `into_iter`.

Since our type `VecExtra` is based on the `Vec` type, and `VecExtra` takes `Vec`
as its sole argument, we can use the `Item` and `IntoIter` types that `Vec` uses
directly.

```
// Implement IntoIterator for VecExtra<T>
impl<T> IntoIterator for VecExtra<T> {
    type Item = <Vec<T> as IntoIterator>::Item;
    type IntoIter = <Vec<T> as IntoIterator>::IntoIter;

    fn into_iter(self) -> Self::IntoIter {
        self.0.into_iter()
    }
}
```

Let's extend our excursion into Iterators a bit more. In order to implement the
`IntoIterator` trait, you must define the type of `Item` and `IntoIter`. The
`type Item = ..` part represents the type of elements that iterator will yield.

Since in Rust, traits can be implemented not only for types, but also for
mutable and immutable references to types, we must also implement the
`IntoIterator` trait for `&VecExtra<T>` and ` &mut VecExtra<T>`. While the
implementation for `Vec<T>` takes ownership of the vector and yields its
elements by value, the implementation for `&Vec<T>` does not take ownership and
yields references to the elements of the vector.

And as before, we need to add `'a` as the lifetime specifier to explicitly
indicate that our code does not use data that has been deallocated.

```
// Implement IntoIterator for &VecExtra<T>
impl<'a, T> IntoIterator for &'a VecExtra<T> {
    type Item = <&'a Vec<T> as IntoIterator>::Item;
    type IntoIter = <&'a Vec<T> as IntoIterator>::IntoIter;

    fn into_iter(self) -> Self::IntoIter {
        self.0.iter()
    }
}

// Implement IntoIterator for &mut VecExtra<T>
impl<'a, T> IntoIterator for &'a mut VecExtra<T> {
    type Item = <&'a mut Vec<T> as IntoIterator>::Item;
    type IntoIter = <&'a mut Vec<T> as IntoIterator>::IntoIter;

    fn into_iter(self) -> Self::IntoIter {
        self.0.iter_mut()
    }
}
```

To put it into a nutshell, the implementation of the trait `IntoIterator`
describes how to iterator over datatype. Therefore, `into_iter` will return the
next element of `T`, `&T` or `&mut T`, depending on the exact context, making it
compatible with functions like `collect()` that require the `IntoIterator` trait.

As always, feel free to check out the HashSet implementation in the [add_hashset
branch](https://github.com/chrysh/quarto_rs/tree/add_hashset) on github.
