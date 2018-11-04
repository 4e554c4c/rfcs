- Feature Name: linked_list_cursors
- Start Date: 2018-10-14
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Many of the benefits of linked lists rely on the fact that most operations
(insert, remove, split, splice etc.) can be performed in constant time once one
reaches the desired element. To take advantage of this, a `Cursor` interface
can be created to efficiently edit linked lists. Furthermore, unstable
extensions like the `IterMut` changes will be removed.

# Motivation
[motivation]: #motivation

From Programming Rust:
> As of Rust 1.12, Rust’s LinkedList type has no methods for removing a range of
> elements from a list or inserting elements at specific locations in a list.
> The API seems incomplete.

Both of these issues have been fixed, but in different and incompatible ways.
Removing a range of elements is possible though the unstable `drain_filter` API,
and inserting elements in at specific locations in a list is possible through
the `linked_list_extras` extensions to `IterMut`.

This motivates the need for a standard interface for insertion and deletion of
elements in a linked list. An efficient way to implement this is through the use
of "cursors". A cursor represents a position in a collection that can be moved
back and forth, somewhat like a `DoubleEndedIterator`. However, mutable cursors
can also edit the collection at their position.

A mutable cursor would allow for constant time insertion and deletion of
elements and insertion and splitting of lists at its position. This would allow
for simplification of the `IterMut` API and a complete LinkedList
implementation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The cursor interface would provides two new types: `Cursor` and `CursorMut`.
These are created in a similar way as iterators.

With a `Cursor` one can seek back and forth through a list and get the current
element. With a `CursorMut` One can seek back and forth and get mutable
references to elements, and it can insert and delete elements before and behind
the current element (along with performing several list operations such as
splitting and splicing).

Lets look at where these might be useful.

## Examples

This interface is helpful most times insertion and deletion are used together.

For example, consider you had a linked list and wanted to remove all elements
which satisfy a certain predicate, and replace them with another element. With
the old interface, one would have to insert and delete separately, or split the
list many times. With the cursor interface, one can do the following:

``` rust
fn remove_replace<T, P, F>(list: &mut LinkedList<T>, p: P, f: F)
where
    P: Fn(&T) -> bool,
    F: Fn(T) -> T,
{
    use list_cursors::StartPosition::BeforeHead;

    let mut cursor = list.cursor_mut(BeforeHead);
    // move to the first element, if it exists
    loop {
        let should_replace = match cursor.peek_after() {
            Some(element) => p(element),
            None => break,
        };
        if should_replace {
            let old_element = cursor.pop_after().unwrap();
            cursor.insert_after(f(old_element));
        }
        cursor.move_next();
    }
}
```

This could also be done using iterators. One could transform the list into an
iterator, perform operations on it and collect. This is easier, however it still
requires much needless allocation.

For another example, consider code that was previously using `IterMut`
extensions.
``` rust
fn main() {
	let mut list: LinkedList<_> = (0..10).collect();
	{
		let mut iter = list.iter_mut();
		while let Some(x) = iter.next() {
			if *x >= 5 {
				break;
			}
		}
		iter.insert_next(12);
	}
	println!("{:?}", list);
	// [0, 1, 2, 3, 4, 5, 12, 6, 7, 8, 9]
}
```
This can be changed almost verbatim to `CursorMut`:
``` rust
fn main() {
	use list_cursors::StartPosition::BeforeHead;
	let mut list: LinkedList<_> = (0..10).collect();
	{
		let mut cursor = list.cursor_mut(BeforeHead);
		while let Some(x) = cursor.next() {
			if *x >= 5 {
				break;
			}
		}
		cursor.insert_before(12);
	}
	println!("{:?}", list);
	// [0, 1, 2, 3, 4, 5, 12, 6, 7, 8, 9]
}
```

Contrary to the `IterMut` interface the algorithm can be easily inverted
by simply replacing `next` with `prev`, `*_before` with `*_after`, and `>=` by `<=`:
``` rust
fn main() {
	use list_cursors::StartPosition::AfterTail;
	let mut list: LinkedList<_> = (0..10).collect();
	{
		let mut cursor = list.cursor_mut(AfterTail);
		while let Some(x) = cursor.prev() {
			if *x <= 3 {
				break;
			}
		}
		cursor.insert_after(12);
	}
	println!("{:?}", list);
	// [0, 1, 2, 12, 3, 4, 5, 6, 7, 8, 9]
}
```

In general, the cursor interface is not the easiest way to do something.
However, it provides a basic API that can be built on to perform more
complicated tasks.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A cursor is created by using one of the following methods of list. For
the raw `cursor`/`cursor_mut` methods the returned cursor would point to
an "empty" element as requested by the position argument:

``` rust
pub enum StartPosition {
    BeforeHead,
    AfterTail,
}
```
Calling `current()` on a cursor in this state would return `None`. The ability
to start form an empty element is nessesary, because a cursor will never pop
the element it is pointing to - therefore to drain the list from either end
it is nessesary to start with an empty element.

The `head()`/`head_mut()`, and `tail()`/`tail_mut()` methods serve as a
convenience methods that attempt to start the iteration from a non-empty
element. This mehods ensure that starting at either end of the
list keeps the cursor at this end or the following/preceding empty element.
For example, the `head*()` and `tail*()` of an empty list are the empty
elements before and after the list. For an one element list both metohds
point to this element. This behavior also ensures that inserting before
head and after tail has the expected semantincs for the movement of the cursor.

``` rust
/// Provides a cursor with immutable references to elements in the list.
/// The cursor starts ether with an empty element before the head or after the tail
/// as specified by  the `position` argument.
pub fn cursor(&self, position: StartPosition) -> Cursor<T> {...}

/// Provides a cursor with immutable references to elements in the list starting with
/// head of the list, i.e. the first valid element or the empty element before the head.
pub fn head(&self) -> Cursor<T> {...}

/// Provides a cursor with immutable references to elements in the list starting with
/// tail of the list, i.e. the last valid element or the empty element after the tail.
pub fn tail(&self) -> Cursor<T> {...}

/// Provides a cursor with mutable references to elements and mutable access to the list.
/// The cursor starts ether with an empty element before the head or after the tail
/// as specified by  the `position` argument.
pub fn cursor_mut(&mut self, position: StartPosition) -> CursorMut<T> {...}

/// Provides a cursor with mutable references to elements and mutable access to the list
/// starting with head of the list, i.e. the first valid element or the empty element
/// before the head.
pub fn head_mut(&mut self) -> CursorMut<T> {...}

/// Provides a cursor with mutable references to elements and mutable access to the list
/// starting with tail of the list, i.e. the last valid element or the empty element
/// after the tail.
pub fn tail_mut(&mut self) -> CursorMut<T> {...}

```

The `Cursor` would provide the following interface:

``` rust
impl<'list, T> Cursor<'list, T> {
    /// Return cursor position where `None` is the empty element before the head,
    /// `Some(0)..Some(list.len())` are valid element positions and `Some(list.len())`
    /// is the empty element after the tail.
    pub fn pos(&self) -> Option<usize> {...}

    /// Move to the subsequent element of the list if it exists or the empty
    /// element after the tail. Returns the element the cursor pointed to last.
    pub fn next(&mut self) -> Option<&'list T> {...}

    /// Move to the previous element of the list if it exists or the empty
    /// element before the head. Returns the element the cursor pointed to last.
    pub fn prev(&mut self) -> Option<&'list T> {...}

    /// Move to the subsequent element of the list if it exists or the empty
    /// element after the tail.
    pub fn move_next(&mut self) {...}

    /// Move to the previous element of the list if it exists or the empty
    /// element before the head.
    pub fn move_prev(&mut self) {...}

    /// Get the current element
    pub fn current(&self) -> Option<&'list T> {...}

    /// Get the following element
    pub fn peek_after(&self) -> Option<&'list T> {...}

    /// Get the previous element
    pub fn peek_before(&self) -> Option<&'list T> {...}
}

```
The `CursorMut` would provide the following interface:

```rust

impl<'list T> CursorMut<'list, T> {
    /// Return cursor position where `None` is the empty element before the head,
    /// `Some(0)..Some(list.len())` are valid element positions and `Some(list.len())`
    /// is the empty element after the tail.
    pub fn pos(&self) -> Option<usize> {...}

    /// Move to the subsequent element of the list if it exists or the empty
    /// element after the tail. Returns the element the cursor pointed to last.
    pub fn next(&mut self) -> Option<&mut T> {...}

    /// Move to the previous element of the list if it exists or the empty
    /// element before the head. Returns the element the cursor pointed to last.
    pub fn prev(&mut self) -> Option<&mut T> {...}

    /// Move to the subsequent element of the list if it exists or the empty
    /// element after the tail.
    pub fn move_next(&mut self) {...}

    /// Move to the previous element of the list if it exists or the empty
    /// element before the head.
    pub fn move_prev(&mut self) {...}

    /// Get the current element
    pub fn current(&mut self) -> Option<&mut T> {...}

    /// Get the next element
    pub fn peek_after(&mut self) -> Option<&mut T> {...}

    /// Get the previous element
    pub fn peek_before(&self) -> Option<&mut T> {...}

    /// Get an immutable cursor at the current element
    pub fn as_cursor(&self) -> Cursor<T> {...}

    /// Insert `element` after the cursor
    ///
    /// This function will move the cursor so `peek_after` will point to the new element.
    pub fn insert_after(&mut self, element: T) {...}

    /// Insert `element` before the cursor
    ///
    /// This function will move the cursor so `peek_before` will point to the new element.
    pub fn insert_before(&mut self, element: T) {...}

    /// Insert `list` between the current element and the next
    ///
    /// This function will move the cursor so `peek_after` will point to the head
	///  of the inserted `list`.
    pub fn insert_list_after(&mut self, mut list: LinkedList<T>) {...}

    /// Insert `list` between the previous element and current
    ///
    /// This function will move the cursor so `peek_before` will point to the tail
	/// of the inserted `list`.
    pub fn insert_list_before(&mut self, mut list: LinkedList<T>) {...}

    /// Remove and return the element following the cursor
    ///
    /// This function will not consume the element pointed to by the cursor. If you
    /// want to drain the list you should start with an empty element.
    pub fn pop_after(&mut self) -> Option<T> {...}

    /// Remove and return the element before the cursor
    ///
    /// This function will not consume the element pointed to by the cursor. If you
    /// want to drain the list you should start with an empty element.
    pub fn pop_before(&mut self) -> Option<T> {...}

    /// Split the list in two after the current element
    /// The returned list consists of all elements following but **not including**
    /// the current one.
    pub fn split_after(self) -> LinkedList<T> {...}

    /// Split the list in two before the current element
    /// The returned list consists of all elements following and **including**
    /// the current one.
    pub fn split_before(self) -> LinkedList<T> {...}
}

```
One should closely consider the lifetimes in this interface. Both `Cursor` and
`CursorMut` operate on data in their `LinkedList`. This is why, they both hold
the annotation of `'list`.

The lifetime elision for their constructors is correct as
```rust
pub fn cursor(&self) -> Cursor<T>
```
becomes
```rust
pub fn cursor<'list>(&'list self) -> Cursor<'list, T>
```
which is what we would expect. (the same goes for `CursorMut`).

Since `Cursor` cannot mutate its list, `current`, `peek` and `peek_before` all
live as long as `'list`. However, in `CursorMut` we must be careful to make
these methods borrow. Otherwise, one could produce multiple mutable references
to the same element.

The only other lifetime annotation is with `as_cursor`. In this case, the
returned `Cursor` must borrow its generating `CursorMut`. Otherwise, it would be
possible to achieve a mutable and immutable reference to the same element at
once.

One question that arises from this interface is what happens if `move_next` is
called when a cursor is on the last element of the list, or the list is empty (or
`move_prev` and the beginning). The proposed solution is to effectively
have an empty element on each end of the list that corresponds to one of the
`StartPosition` variants. Moving next to the `AfterEnd` is a no-op as is the
moving previous to `BeforeEnd`. Moving to the opposite direction yields
the first nonempty element or the opposite empty element. Internally the state
is recorded in a `Position` enum, that admits only these valid states.

```rust
enum Position<T> {
    BeforeHead,
    Node(usize, NonNull<Node<T>>),
    AfterTail,
}
```

A large consequence of this new interface is that it is a complete superset of
the already existing `Iter` and `IterMut` API. Therefore, the following two
methods added to `IterMut` in the `linked_list_extras` features should be
removed or depreciated:
- `IterMut::insert_next`
- `IterMut::peek_next`
The rest of the iterator methods are stable and should probably stay untouched
(but see below for comments).

# Drawbacks
[drawbacks]: #drawbacks

The cursor interface is rather clunky, and while it allows for efficient code,
it is probably not useful outside of many use-cases.

One of the largest issues with the cursor interface is that it exposes the exact
same interface of iterators (and more), which leads to unnecessary code
duplication.
However, the purpose of iterators seems to be simple, abstract and easy to use
rather than efficient mutation, so cursors and iterators should be used
in different places.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are several alternatives to this:

1. Implement cursors as a trait extending `Iterator` (see the cursors
pseudo-rfc below)

Since the cursors are just an extension of iterators, it makes some sense to
create them as a trait. However, I see several reasons why this is not the best.

First, cursors work differently than the existing `Iterator` extensions like
`DoubleEndedIterator`. In a `DoubleEndedIterator`, if one calls `next_back` and
then `next`, it should not return the same value, so unlike a cursor, a
`DoubleEndedIterator` does not move back and forth throughout a collection.

Furthermore, while `Iterator` is a general interface for many collections,
`Cursor` is very much specific to linked lists. In other collections such as
`Vec` a cursor does not make sense. So it makes little sense to make a trait
when it will only be used in one place.

2. Using the `IterMut` linked list extensions

Insertion was added to `IterMut` in the `linked_list_extras` feature. Many of
these features could be added to it just as well. But, this overcrowds `IterMut`
with many methods that have nothing to do with iteration (such as deletion,
splitting etc.)
It makes sense to put these explicitly in their own type, and this can be
`CursorMut`.

3. Do not create cursors at all

Everything that cursors do can already be done, albeit in sometimes a less
efficient way. Efficient code can be written by splitting linked lists often,
and while this is a complicated way to do things, the rarity of the use case may
justify keeping things how they are.

# Prior art
[prior-art]: #prior-art

- [cursors pseudo-rfc](https://internals.rust-lang.org/t/pseudo-rfc-cursors-reversible-iterators/386/18)

This rust internals post describes an early attempt at making cursors. The
language was in a different state when it was written (pre-1.0), so details have
changed since then. But this describes several different approaches to making
cursors and where they led.

- Java-style iterators

Java (and other languages) tried to fix this by adding a `remove` function to
their iterators. However, I feel this method would not be the best choice for
Rust (even for specific `IterMut`s like those in LinkedList) since it diverges
from the expected behaviour of iterators.

- [linked list extras issue](https://github.com/rust-lang/rust/issues/27794)

Discussion on the issue tracker about how this is currently managed with
modifications to `IterMut`. The consensus seems to be that it is incomplete, and
it is suggested to create a new `Cursor` and `CursorMut` types.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How will this interface interact with iterators?

Will we keep both `Iter` and `Cursor` types? Implement one with another? I feel
like they should be different things, but there is reason to consolidate them.

- Only for linked lists?

Should we implement this for more collections? It could make sense for other
collections, such as trees and arrays, but the design would have to be reworked.
