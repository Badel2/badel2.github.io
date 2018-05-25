---
layout: post
title: Tuenti Challenge 8
---

An explaination of my solutions to the #tuentichallenge8

# Introduction

First of all, a huge thank you to all the organizers. 

In this post I talk about my experience and explain some parts of the code, I
assume the reader is familiar with the challenges, so keep this tab open:

<https://contest.tuenti.net/>

My code is available here:

<https://github.com/Badel2/TuentiChallenge8>

If you have any questions or comments,
[join the discussion on reddit](https://www.reddit.com/r/Badel2/comments/8ljf9x/tuenti_challenge_8/)
.

And since this turned out pretty long, if you are only interested in some
specific challenges use this links to jump to them:

[P01](#p01)
[P02](#p02)
[P03](#p03)
[P04](#p04)
[P05](#p05)
[P06](#p06)
[P07](#p07)
[P08](#p08)
[P09](#p09)
[P10](#p10)
[P11](#p11)
[P12](#p12)
[P13](#p13)
[P14](#p14)
[P15](#p15)

# P01

### Waffles

The logic for this challenge is pretty straight-forward, so I will explain my
setup instead. I use debian as the operating system and Rust as the programming
language. In all the challenges I read the input from stdin and print the
output to stdout. To compile and run the code I run a command similar to this:

```sh
rustc p01.rs
./p01 < testInput > testOutput
```

Or just

```sh
cargo run -- < testInput > testOutput
```

This is the code that reads the input. It may seem quite verbose compared to
other languages, and indeed it is.

```rust
use std::io;

fn main() {
    let mut input = String::new();
    io::stdin().read_line(&mut input).unwrap();
    let num_cases: u32 = input.trim().parse().unwrap();

    for i in 0..num_cases {
        input.clear();
        io::stdin().read_line(&mut input).unwrap();
        let nm: Vec<u32> = input
            .trim()
            .split(" ")
            .map(|s| s.parse::<u32>().unwrap())
            .collect();

        let n = nm[0];
        let m = nm[1];
        println!("Case #{}: {}", i + 1, waffles(n, m));
    }
}
```

Notice all the calls to `unwrap()`? In a real application they would be
replaced with proper error handling: reading from stdin can fail, parsing a
string as an integer can fail. Instead of throwing exceptions, Rust favors
explicit error handling.

This is the equivalent to `getline`, we need to clear the input
because any new data is appended to the input string.

```rust
input.clear();
io::stdin().read_line(&mut input).unwrap();
```

Parsing a line with only one integer is easy, just annotate the type somewhere:

```rust
let x: u32 = input.trim().parse().unwrap();
let x = input.trim().parse::<u32>().unwrap();
```

But parsing multiple elements per line is a bit more complicated, we need to
split the input string, parse every part, and store it into a vector.

```rust
let nm: Vec<u32> = input
    .trim()
    .split(" ")
    .map(|s| s.parse::<u32>().unwrap())
    .collect();
```

Then we just index this vector to get the desired input, which is pretty ugly:

```rust
let n = nm[0];
let m = nm[1];
```

There is an experimental slice pattern syntax (stable since Rust 1.26) which
would transform that into:

```rust
let mut arr = [0; 2];
arr.copy_from_slice(&nm);

let [n, m] = arr;
```

Which is slightly nicer as long as you ignore the first two lines.

All this code can probably be replaced with a Python one-liner, so solving this
simple challenges in Rust is just personal preference.

# P02

### Nazi base

I usually use `cargo` to manage my Rust projects, but when a challenge can be
solved without using any external libraries, I just submit the `main.rs` file.
In this challenge I had to use the
[num\_bigint](https://github.com/rust-num/num-bigint)
crate because we need to be able to calculate up to about `26^26`, and 64 bits
are not enough. However 128 bits would have been enough, and Rust has support
for 128 bit integers since version 1.26. The funny thing is that at the time of
the challenge, the stable Rust version was 1.25, so I decided not to use these
nightly features. As a result, the code is slightly more verbose.

See
[blog/p02\_u128/](https://github.com/Badel2/TuentiChallenge8/blob/master/blog/p02_u128/p02_u128.rs)
for the exact same code using u128.

The solution depends only on the input string length, so I precalculated all
the possible values into a lookup table.

The algorithm is pretty simple, it
just calculates the maximum value (543210) and the minimum value (102345) and
returns the difference.

```rust
let mut min = 0;
let mut max = 0;
let mut p = 1;
for x in 0..base {
    max += x * p;
    // No leading zeros: transform 012 into 102
    min += match base - x - 1 {
        0 => 1,
        1 => 0,
        n => n,
    } * p;
    p *= base;
}

return max - min;
```

For the minimum value we need to swap `0` with `1`, so `012` becomes `102`,
because `012` is not a valid number (it cannot have leading zeros). This is
accomplished using Rust's match, which returns a value so we can inline it like
that. The equivalent of that match in C would be:

```c
n = base - x - 1;
min += (
        (n == 0) ? 1 :
        (n == 1) ? 0 :
         n
       ) * p;
```

No, wait, that's too readable, how about:

```c
min += (n<2?1-n:n) * p;
```

Nice.

# P03

### Scales and notes.

The algorithm is pretty simple:

* Create all 24 possible scales
* Remove duplicate notes from input
* For each note: remove all the scales which don't contain this note

And we are left with all the scales that contain all the notes. 

```rust
fn all_scales_containing(n: &[Note]) -> String {
    let mut a = Scale::all_scales();
    // We make a copy of n because we want to modify it
    let mut uniq_notes = n.to_vec();
    uniq_notes.sort();
    uniq_notes.dedup();

    for ref n in uniq_notes {
        a.retain(|ref s| s.notes.contains(n));
    }
    ...
```

The code is simple thanks to the `retain` function, which iterates over the
vector and removes all the elements for which the predicate returns false.
`contains` is also a builtin function, which simply searches an element in a
vector. There are many useful functions which greatly simplify the code,
sometimes it's a pain to work without them.

As for the parsing code, `Note` is just an enum:

```rust
enum Note {
    A,
    As,
    B,
    C,
    Cs,
    ...
```

And then we have a giant match statement to match strings to notes, merging
equivalent notes to the sharp version:

```rust
match s {
    "A" => A,
    "A#" => As,
    "Bb" => As,
    "B" => B,
    "B#" => C,
    "Cb" => B,
    "C" => C,
    "C#" => Cs,
    "Db" => Cs,
    ...
```

Before creating all the scales we need to create all the notes, and since
Rust is a type safe language we can't just do

```c
for(i=0; i<12; i++) {
    Note n = (Note)i;
    notes.push(n);
}
```

There exist crates which provide that functionality, but the
workarround is just to copy the enum definition and add `vec![` before it, like
this:

```rust
fn all_notes() -> Vec<Note> {
    vec![
        A,
        As,
        B,
        C,
        Cs,
        ...
```

Although it is valid to do the opposite conversion, for example to check if
a scale is a major scale we cast the `Note` to `u8`:

```rust
let major = ((12 + self.notes[2] as u8 - self.notes[1] as u8) % 12) == 2;
```

A simple test to verify that the code works is to print all the scales, which can be accomplished with this one liner:

```rust
println!("{:#?}", Scale::all_scales().iter().map(|s| (s.name(), s.notes.clone())).collect::<Vec<_>>());
```

Which gives us a nicely formatted output:
```rust
[
    (
        "MA",
        [
            A,
            B,
            Cs,
            D,
            E,
            Fs,
            Gs
        ]
    ),
    (
        "MA#",
        [
            As,
            C,
            D,
            Ds,
            F,
            G,
            A
        ]
    ),
    ...
```

# P04

### Knight Princess Exit

This challenge consists of finding the shortest path from the knight to the
princess and then to the exit. I don't know about you, but when I hear _the
shortest_ path I immediately think Dijkstra. So yeah, if we can transform the
map into a graph we're done. There are two paths to calculate but we can treat
them as two independent problems since in the worst case we will have to
calculate everything twice, which isn't a big deal.

I opted for generating the edges on demand. We store the map as a
`Vec<Material>`, we index it using a 2D `Pos { x, y }`, using the simple
`x + y * width` formula to get the element from the 1D vector. The important
function is `get_edges` which returns all the possible jumps that the knight
can make, depending on the material of the current square: ground, trampoline
or lava (there are no possible jumps when on lava, symbolizing the inevitable
death). Since there are only 8 possible jumps I just quickly copy-pasted them
all:

```rust
pos.push(Pos { x: p.x + 2, y: p.y + 1, });
pos.push(Pos { x: p.x + 2, y: p.y - 1, });
pos.push(Pos { x: p.x - 2, y: p.y + 1, });
pos.push(Pos { x: p.x - 2, y: p.y - 1, });
pos.push(Pos { x: p.x + 1, y: p.y + 2, });
pos.push(Pos { x: p.x + 1, y: p.y - 2, });
pos.push(Pos { x: p.x - 1, y: p.y + 2, });
pos.push(Pos { x: p.x - 1, y: p.y - 2, });
pos.retain(|&p| p.x < self.size.0 && p.y < self.size.1);
```

Finally I had the idea to remove jumps which leave the knight outside the
board, but that optimization is not really needed since we return "no edges"
when called with a position outside the board anyway. However, since the
position is stored as an unsigned integer (u32), when we substract from it we
can get a negative number, which will overflow. In this case that won't be a
problem because `-1` is `2^32 - 1` which is well outside the board anyway, but
Rust has overflow checks enabled by default in debug mode, so it will panic at
runtime if there is an overflow. One solution would be to explicitly state that
we want to use wrapping arithmetic operations:

```rust
pos.push(Pos { x: p.x.wrapping_add(2), y: p.y.wrapping_sub(1), });
```

But I didn't want to rewrite that 16 lines of code, so I just compiled in
release mode, which disables overflow checks.

As for the Dijkstra algorithm, I just copied it from
[here](https://doc.rust-lang.org/std/collections/binary_heap/)
with a small change to use a `HashMap` instead of a `Vec` for keeping track
of the minimum cost of each node.

# P05

### DNA slices

In this challenge we need to connect to a server. This is the base code which
starts a TCP connection and enters a read loop:

```rust
use std::io::{Read, Write};
use std::net::TcpStream;

fn main() {
    let mut stream = TcpStream::connect(("52.49.91.111", 3241)).unwrap();
    let mut buffer = vec![0u8; 1024];

    //let state = b"TEST\n";
    let state = b"SUBMIT\n";

    'tcp: loop {
        match stream.read(&mut buffer) {
            Ok(0) => break 'tcp,
            Ok(n) => {
                let buf = &buffer[0..n];
                let s = String::from_utf8_lossy(&buf);
                println!("{}", s);
                if let Some(_) = find_subsequence(&buf, br#"> Please, provide "TEST" or "SUBMIT""#)
                {
                    stream.write(state).unwrap();
                } else if buf[0] != b'>' {
                    // DATA
                    //[...]
                }
            }
            Err(e) => panic!("Error reading stream: {:?}", e),
        }
    }
}
```

The `buffer` is just a 1024-byte array, when we read from the stream we don't usually
fill the buffer so we alias the first `n` bytes:

```rust
let buf = &buffer[0..n];
```

Both `buffer` and `buf` work on bytes, not on chars. The first this I do is convert
the `buf` into a `String`, and print it. This is important because we want to save
the token when we get it!

We can use the `find_subsequence` helper function to search for some bytes in
buf. We use that to write "TEST" or "SUBMIT" when asked to.

We work on `Vec<u8>` instead of `String` because Rust has proper unicode
support so we can't index strings, and even something as simple as a `strlen`
becomes O(n), because number of bytes `!=` number of chars. This is why the
literals begin with a `b`: `b"TEST\n"` is a byte array, not a string.

I assumed that the lines which do not begin by `>` are DNA parts.
We parse them using the usual `trim().split(" ")` combo but this time we also
add `.enumerate()`, to keep track of the index of each part, since we need that
for the result:

```rust
// DATA
let parts: Vec<_> = s.trim()
    .split(" ")
    .enumerate()
    .map(|(i, s)| Part::new(i + 1, s.as_bytes().to_vec()))
    .collect();

println!("PARTS: {:#?}", parts);

let idx = find_original_samples(&parts);
println!("Result: {}", idx);
stream.write(idx.as_bytes()).unwrap();
```

Notice that here we get the parts from a `String`, but convert to `Vec<u8>`
using `s.as_bytes().to_vec()`. Similarly, the result of `find_original_samples`
is also a `String`, and we must convert it to `&[u8]` to send it through the
stream. When I was a beginner stuff like this was a pain. Now it still is a
pain but I'm used to it.

It took me a while to understand the question: there is
an original sample, which is repeated twice:

```text
aaccGgTTC aaccGgTTC
```

Then this two samples are randomly split into parts:

```text
aacc GgTTC aaccG gTTC
```

Then some random parts are added to the mix, and the whole thing is shuffled:

```text
aacc GgTTC aaccG gTTC aaaa cccc gttc
```

Now our task is to find which parts are from the original sample. Since the two
original samples are randomly split we can't just compare the parts, however we
know that the beginning and end of each sample will be intact, so we can
compare that. For example, we take the first part and check if any other part
shares a common start with it:

```text
aacc
GgTTC aaccG gTTC aaaa cccc gttc
```

Then, we take the remaining letters (in this case the last `G` from `aaccG`),
and repeat the process until no parts match:

```text
G
GgTTC gTTC aaaa cccc gttc

gTTC 
gTTC aaaa cccc gttc

aaaa
cccc gttc
```

Then, the original parts are the ones that we removed from the list.

Now back to code:

<!--
Wtf I accidenally pressed Ctrl-N and discovered that vim has autocomplete
support. For some reason the top word is "rust".
-->

There is a helper function which checks if two parts share a common start,
and returns the remaining letters, or `None` if they don't match:

```rust
fn share_common_start(a: &[u8], b: &[u8]) -> Option<Vec<u8>> {
    // Make sure a is the longest string:
    let (a, b) = if a.len() > b.len() { (a, b) } else { (b, a) };
    if a[0..b.len()] == *b {
        Some(a[b.len()..a.len()].to_vec())
    } else {
        None
    }
}
```

For example:

```text
a: TTCGGTCT
b: TTCG
returns Some(GTCT)
```

The main loop calls `find_original_samples` which takes care of the output
format, but the good stuff is in the `find_samples` function.

```rust
fn find_samples(start: &[u8], p: &[Part]) -> Option<Vec<usize>>
```

It takes the start substring, a list of parts, and returns the indexes of the
parts which are from the original sample. When called with an empty `start`, it
will iterate over all the parts trying each of them as a start. If it finds a
match, it will recursively find all the possible "original sample parts". Here
is the code for the case where the start is not empty:

```rust
for x in p {
    if let Some(new_start) = share_common_start(&x.data, start) {
        let mut rem_p = p.to_vec();
        // Remove x from the remaining parts
        rem_p.retain(|t| t != x);
        if let Some(indexes) = find_samples(&new_start, &rem_p) {
            let mut ix = indexes;
            // Add x to the original sample index list
            ix.push(x.idx);
            return Some(ix);
        }
    }
}

return None;
```

It's a nice recursive algorithm. I got the idea from a similar problem, the
[merged string puzzle from the datagenetics blog](https://datagenetics.com/blog/april12013/index.html)
, which is a really nice blog and you should definitely check it out.

# P06

### Button Hero

This was the first problem where I had to rewrite my program after passing
the test case, because I couldn't even solve the first submit case.

To begin, it is important to realize that we don't care about the position and
speed of the notes, we only need the time when we must press and release the
button. The transformation is simple:

```rust
let t0 = x / s;
let t1 = (x + l) / s;
```

We will store these values in a struct called `Interval`:

```rust
struct Interval {
    t0: u32,
    t1: u32,
}
```

My original algorithm consisted of a divide and conquer approach where I split
the map into non-overlapping clusters and solve each cluster independently. By
solving each cluster I mean trying all the possible combinations: press one
note (and remove all the notes which overlap with it), this will create new
clusters which will be solved recursively, until there are no more notes
remaining. In the end we just return the maximum score.

The new and working algorithm builds a graph with the time intervals. For each
interval, its children are the intervals which can be pressed after that one,
with the restriction that if we can press a note for free, we must take it.
Consider the following example:

```rust
.AAA............
......BBBBBBBB..
.......CCC......
...........DDD..
      ^      ^
      B.t0   B.t1
```

Here if we take as A's children all the notes with `t0 < B.t1`, we will have 3
paths: AB, AC, AD. However the AD path would be wasteful because we can take
the ACD path. So we don't consider D to be A's child. We will see the code
later.

First we build a `Map`, which just stores all the intervals ordered by `t0`.

```rust
struct Map {
    score: u32,
    intervals: BTreeMap<Interval, u32>,
}
```

We use a `BTreeMap` (ordered\_set) because it provides the `range()` function,
so we can easily iterate over `Interval`s based on restrictions on `t0`. The
score field used to store the extra score gained for pressing trivial notes,
the ones which do not overlap with anything else. But we don't use that
optimization in the graph approach. Thanks to the entry API we can build the
`BTreeMap` while merging all the identical intervals, two notes A and B with
the same `t0` and `t1` will be merged as a note N with score A+B:

```rust
let mut intervals = BTreeMap::new();
for n in notes {
    let a = Interval::from_note(&n);
    *intervals.entry(a).or_insert(0u32) += n.p;
}
```

Usually working with graphs in Rust is a pain because of the ownership rules.
Luckly, in this case the graph doesn't have any cycles, so it's not that bad.
Each node can have multiple children, which isn't a problem, but each child can
have multiple parents. With multiple parents we need reference counting to make
sure that there is no node with a reference to this child when deallocating it.
This is one of the main selling points of Rust, to ensure memory safety and
forget about invalid pointers! Aside from reference counting, there is also the
arena approach where an arena owns all the nodes, so they are all deallocated
at once, and the nodes are accessed using indexes to the arena instead of
pointers. But I wanted to try the reference counting approach, so our nodes
will look like this:

```rust
struct Node {
    t: Interval,
    score: u32,
    children: Vec<Rc<Node>>,
}
```

`Rc<Node>` is a reference-counting pointer to `Node`, just like a
`std::shared_ptr` in C++.

To build our graph we need to convert the `Interval`s into `Rc<Node>`, but once
we insert the `Rc<Node>` into the graph we cannot mutate it. That's another
selling point of Rust, if a resource is shared, it can not be mutated. This
means that we can only add children before adding a node to the graph. I guess
it should be possible to create the graph in reverse order, starting at t0=max
and ending at t0=0, but I wanted to start at t0=0. So, when Rust doesn't allow
you to mutate something, you just throw a `RefCell` at it:

```rust
struct Node {
    t: Interval,
    score: u32,
    children: RefCell<Vec<Rc<Node>>>,
}
```

A `RefCell` is basically a way to enforce the "no mutability when shared" rule
at runtime. It will only let you mutate the inner value if nobody else is using
it.

Let's start building the graph. First we convert the intervals into `Rc<Node>`.

```rust
// self.intervals: BTreeMap<Interval, u32>
let rcn: HashMap<_, _> = self.intervals
    .iter()
    .map(|(i, s)| (i, Rc::new(Node::new(*i, *s))))
    .collect();
```

For some reason I decided to store the `Rc<Node>`s in a `HashMap<Interval,
Rc<Node>>`. This way we can work with `self.intervals`, and index `rcn` to
get the corresponding `Rc<Node>` for each `Interval`.

Next we create the root node, which is just an interval with t0=0. Then,
we take the interval with the lowest t0 (the first note to be pressed)
and try to find the minimum t1 over all the intersecting notes.

```rust
let mut root = Node::root();

let (first, _first_s) = self.intervals.iter().next().unwrap();
let mut true_limit = first.t1;
for (x, _s) in self.intervals.range(first..) {
    true_limit = min(true_limit, x.t1);
    if x.t0 > true_limit {
        break;
    }
}
```

That's the optimization I was talking about before, here the `true_limit`
for A would be C.t1, and A's children would be B and C:

```rust
.AAA............
......BBBBBBBB..
.......CCC......
...........DDD..
         ^
         C.t1
```

This is an optimization which I accidentally didn't implement, because in my
code I wrote `if x.t1 > true_limit` (facepalm). But I thought it would be
necessary, and the difference in runtime is pretty noticeable: 1.429s vs
0.517s on submitInput.

Once we have the `true_limit`, we want to add all these nodes as children:

```rust
let mut children = vec![];
let limit_up = Interval {
    t0: true_limit,
    t1: u32::MAX,
};
for (x, _s) in self.intervals.range(first..&limit_up) {
    children.push(Rc::clone(&rcn[x]));
}

root.ch = RefCell::new(children);

root
```

The function returns the root node.
Once we have the graph we just need one simple function:

```rust
fn score_plus_max_of_children(&self) -> u32 {
    let ch = self.ch.borrow();
    let mut mc = 0;
    for n in ch.iter() {
        let a = n.score_plus_max_of_children();
        mc = max(mc, a);
    }

    self.s + mc
}
```

This function has a pretty descriptive name, it returns the score of the node
plus the score of its best child, plus the score of the best child of its best
child, keep going until a node has no children. When called on the root node,
it will return the maximum score for this graph. However we are missing something
very important, to avoid calculating the score of a node multiple times we need
to cache the result! Without a cache we can't even calculate the first case from
submitInput.

> Recursion is key, but recursion + cache is god.
>
> -- Me after completing P10

```rust
struct Node {
    t: Interval,
    s: u32,
    ch: RefCell<Vec<Rc<Node>>>,
    max_of_children: RefCell<Option<u32>>,
}
```

And again, since a node can be shared by multiple parents, we need a `RefCell`
to introduce mutable state. Well, in this case I should have used a `Cell` but
I was too lazy to add `use std::cell::Cell;`.

```rust
// Using RefCell
fn score_plus_max_of_children(&self) -> u32 {
    // Use cached result when possible
    if let Some(s) = &*self.max_of_children.borrow() {
        return self.s + s;
    }

    let ch = self.ch.borrow();
    let mut mc = 0;
    for n in ch.iter() {
        let a = n.score_plus_max_of_children();
        mc = max(mc, a);
    }

    *self.max_of_children.borrow_mut() = Some(mc);

    self.s + mc
}
```

And with this small change our code completes the submitInput in less than a second!
Recursion is truly magic.

### RefCell vs Cell

But going back to `RefCell` vs `Cell`:
The difference is that `RefCell`
has a reference count and it won't let you modify the value if it is already
borrowed by someone else, while `Cell` just gives you a copy of its value, so
it has zero overhead. ~~But `Cell` cannot be used with vectors, or anything
that involves allocation because it needs a `Copy` type.~~
Wait, it looks like `Cell` can be used on any type now, (although we will lose
the ability to `Debug` and `Clone` nodes) so let's change our code to use `Cell`
instead and see how big is that overhead:

```
RefCell: 0.517s, Cell: 0.476s
```

Well, it's something. The main difference is that when using `RefCell` the
children vector always holds a valid value, but with the `Cell` we literally
take the vector when we need it, leaving an empty vector in its place, and we
need to put it back later:

```rust
// Using Cell
fn score_plus_max_of_children(&self) -> u32 {
    // Use cached result when possible
    if let Some(s) = self.max_of_children.get() {
        return self.s + s;
    }

    let ch = self.ch.take();
    // self.ch is now an empty vector
    let mut mc = 0;
    for n in ch.iter() {
        let a = n.score_plus_max_of_children();
        mc = max(mc, a);
    }
    self.ch.set(ch);
    // self.ch is set to its original value

    self.max_of_children.set(Some(mc));

    self.s + mc
}
```

Notice that the "empty value" problem doesn't exist with `max_of_children`,
because we can safely copy that type (`Option<u32>`) so when we need to read
the value we just call `get()`, which returns a copy.

<!--
So there is a small window where a node seems to have no children, because we
leave an empty vector. Notice that this problem doesn't exist with
`max_of_children`, because we can safely copy that type (`Option<u32>`) so when
we need to read the value we just call `get()`, which returns a copy. Maybe if
we had some bizarre configuration where a node is its own child, then this code
would silently fail, while the `RefCell` code would enter an infinite loop, as
expected. A workarround could be to set the `Cell` to a copy of the original
value immediatly after the take:

```rust
let ch = self.ch.take();
self.ch.set(ch.clone());
```

But cloning a `Vec<Rc<_>>` also has some overhead: it must
increase the reference count of each element! And then, when the old vector
goes out of scope it must decrease the refcount again! I run a quick test:

```
0.517s vs 0.502s vs 0.476s (RefCell, Cell+clone, Cell)
```
-->

You can check out the `Cell` version, where I also fixed the `x.t1 >
true_limit` bug, here:
[blog/p06\_fixed](https://github.com/Badel2/TuentiChallenge8/blob/master/blog/p06_fixed/p06_fixed.rs)

# P07

### GameBoy ROM

I had already downloaded a
[GameBoy debugger](http://bgb.bircd.org)
and began dissassembling the ROM.
I was looking for some strange function with a decryption mechanism that would
be activated with a certain button combination. But then I realized that the
"encrypted" string has a "=" at the end. I don't want to spoil the solution,
look at the code if you are curious. But how did I know that the unknown file
was a ROM, and how did I extract the string? With these useful commands:

```
$ file unknown
unknown: Game Boy ROM image: "UNKNOWN" (Rev.01) [ROM ONLY], ROM: 256Kbit

$ xxd unknown | less
$ strings unknown -n 100 > testInput
```

# P08

### Doors

We sum the door index to its offset so the problem becomes finding the instant
when all the doors are open at once.

We can easily merge two doors: if door A opens every 3 seconds and door B opens
every 5 seconds, then they are equivalent to a door C which opens every 15
(3\*5) seconds. And if we can merge two doors we can merge all the doors, so
the solution for this challenge will just be the offset at which that super-door
opens!

But how to calculate the offset of the merged door? I had no clue, so I just
bruteforced it. It worked for the test case, but it was pretty slow for the
submit case. It took exactly 30 minutes to calculate case 100, and I was
missing 50 cases so I calculated the expected runtime to be about 20 hours, or 5
hours if I ~~implement multithreading~~ split the input into 4 parts and run
the program multiple times at once.

So I left one thread running while I think about a better solution.

Let's start with a few simple cases. We will add an offset to both doors so
that door A opens at t=0. Then, if door B also opens at t=0, door C (A+B) will
open at t=0. If instead, door B opens at t=1, then both doors will coincide at
t=6 (0+3+3, 1+5). Let's see all the cases:

<pre class="font: monospace;">
0123456789012345
<mark>A</mark>..<mark>A</mark>..<mark>A</mark>..<mark>A</mark>..<mark>A</mark>..A..A..A..A..A..A
<mark>B</mark>....B....B....B....B....B....B
.B....<mark>B</mark>....B....B....B....B....
..B....B....<mark>B</mark>....B....B....B...
...<mark>B</mark>....B....B....B....B....B..
....B....<mark>B</mark>....B....B....B....B.
0, 6, 12, 3, 9
</pre>

The doors coincide at t: 0, 6, 12, 3, 9. Notice any pattern? It's 0+6+6-9+6.
Most of the terms just add 6 to the previous one, expect there is a jump from
12 to 3. 12+6 = 18, and both doors are also open at t=18. But since we only
look at the first time both doors are open, and everything repeats every 15
seconds, we must substract 15, or even better: use the modulus operator: `18 %
15 == 3`.

So there is an unknown number, which we will call `tt`. The formula for the
time when both doors are open is:

```c
t = a.offset + tt * b.offset
```

For A=3 and B=5, tt=6. Now, what is 6? 6 is 3\*2, and also 5+1. I
repeated the experiment with other values, using a sheet of paper, and got the
following table:

```c
A B tt
3 5  6
5 3 10
4 5 16
5 4  5
3 7 15
7 3  7 
```

We can see a pattern: `tt = A*n` and `tt = B*m + 1`. In other words,
`tt % A == 0` and `tt % B == 1`.

That's basically the definition of
[modular multiplicative inverse](https://en.wikipedia.org/wiki/Modular_multiplicative_inverse)
:

```
The modular multiplicative inverse (modulo M)
    of an integer a
    is an integer x
such that:

ax = 1 (mod M)
```

If we combine that definition with our equations:

```
a*x % M == 1

tt % B == 1
tt % A == 0

A=a
B=M
tt=a*x
```

Therefore, we just need to find x: the modular multiplicative inverse of A modulo B,
and tt is just `tt = A * x`.

Here I didn't show any Rust code because it doesn't look exactly nice. Since I
had to use BigInts again, and this time the task was not trivial, the code
became pretty messy pretty fast. For example, we must add an ampersand (`&`)
when doing any operation with BigInts because otherwise the BigInt is literally
moved away and doesn't exist under the old name anymore:

```rust
let mut t = &a.p - &a.t;
let mut steps = &p / &a.p;
```

Also, I forgot to consider the case where two doors share a common factor: A=6
B=10 repeat every 30 seconds, which I got right, but the value of `tt` is not
the one we get using the modular inverse formula. Luckly I could just fallback
to the old bruteforce algorithm in that case, and it worked better than I
expected.

I have to admit that at some point I thought about just switching to Python,
because this challenge can be solved applying a few mathematical formulas so
there is little benefit in using Rust.

# P09

### Unscramble a photo.

This was fun, because I had an algorithm in mind but it didn't work at all, it
was a complete failure. My idea was to sort the columns to minimize the
difference between one column and its surroundings. This difference was
calculated as the sum of the differences in color between one pixel (x=0, y=0)
and the corresponding pixel in the next column (x=0, y=1):

```rust
fn similarity(&self, a: &Vline) -> f64 {
    let mut x = 0.0;
    for i in 0..self.pixels.len() {
        x += pixel_diff(&self.pixels[i], &a.pixels[i]);
    }

    x
}
fn pixel_diff(a: &Rgb<u8>, b: &Rgb<u8>) -> f64 {
    let mut x = 0.0;
    for (ca, cb) in a.channels().iter().zip(b.channels()) {
        let ca = *ca as f64;
        let cb = *cb as f64;
        x += f64::abs(ca - cb);
    }

    x
}
```

But as I said, it didn't work as expected, the image was indistinguishable
from the scrambled one. But don't worry, I always have more ideas. The next
idea was to sort the image using diagonal lines as guides. In the test image
there are a few obvious ones just below the QR Code. This is the result of
sorting by of one of these lines (I forgot which one):

![Imgur](https://i.imgur.com/YJ5mxNC.png)

Also, check out this short animation where I try to find an optimal threshold:

<https://imgur.com/tkAscEj>

As you can see, this almost works. At that point I could have tried to improve
my algorithm, perhaphs looking for multiple guide-lines, or doing some
color-based clusterization, but since I could almost see the QR code and I heard
that these codes have nice error correction, I tried to scan it. It didn't work.
But I thought that I can manually reconstruct the missing bits, since the image
is almost sorted anyway, and I have seen
[people reconstructing a QR code from less than that](https://medium.freecodecamp.org/lets-enhance-how-we-found-rogerkver-s-1000-wallet-obfuscated-private-key-8514e74a5433)
. So I decided to make a tool to help me sort the image
line by line.

I used
[SDL2](https://www.libsdl.org/)
since I already have some experience with it, and the
[Rust bindings](https://github.com/Rust-SDL2/rust-sdl2)
are pretty nice. 

The tool consists of a window with the image on the background and a black
vertical line which represents the current selection. That black line is only
aligned with the image when the image width is equal to the window width. I
would post a screenshot, but there's nothing to show, really. All the commands
are performed using the keyboard:

`A` and `D` move the selection left or right.

`W` and `S` increase or decrease the movement speed.

`H` and `L` move the selected column left or right.

`Space` toggles show/hide selection.

`F` toggles faster movement.

`R` rotates the columns of the image.

`Q` sorts the image by diagonal lines, pressing it multiple times changes the y
offset where it looks for lines.

And very important: at any step that modifies the image, we save it and load it
again. That's just because I was too lazy to save the image in RAM. But as a
side effect now we have all these cool videos! :D

Check out this album, it shows what happens when we press `Q` multiple times:

<https://imgur.com/a/w6dBv56>

On the submit image we start with almost half of the QR code!

![Imgur](https://i.imgur.com/gIW8kOM.png)

I have some videos of the manual reconstruction process, the submit one is the
longest with 1:06 minutes and only 10.3 MiB, even though it is made of 3962 PNG
images with a total size of 18.8 GiB, video compression is truly amazing.

You can watch them on YouTube, because Imgur doesn't allow videos longer than
30 seconds. I recommend fullscreen and the highest resolution:

[testImage](https://youtu.be/TFEdgWs8uOc)

[submitImage](https://youtu.be/-mSbEOWoqZM)

In the submit image I was missing two corners in the end, so I just painted
them using GIMP:

![Imgur](https://i.imgur.com/4y1kh8D.png)

Total time spent was 1 hour for the test case and 3 hours for the submit case.

# P10

### Dance

First let's consider the case with no grudges, how many ways are there to make
pairs such that no lines cross? Obviously we can't make pairs if the number of
people is odd.

```
P   N
2   1
4   2
6   5
8   ?
```

P=8 is not trivial, so enjoy my ascii drawing skills: The top left pixel is 0
and we number the people clockwise.

```
 * *     *-*     * *     * *    /* *
*   *   *   *   * \ *   *|  *   *   *
*   *   *   *   *  -*   *|  *   *   *
 * *     * *     * *     * *     * *
```

We can split the P=8 case into 4: if we pair 0 and 1, we are left with 6 people
and we already know that there are 5 ways to pair them. Next we pair 0 and 3,
which leaves 2 people on the right and 4 people on the left. We also know how
to solve these cases. The last two partitions are just reflections of the first
two.

```
N(8) = N(6) + N(2) * N(4) + N(4) * N(2) + N(6)
N(8) = 5 + 1*2 + 2*1 + 5 = 14
```

Now that we have a few numbers, let's google the sequence to see if it matches
something: `1, 2, 5, 14` results in <https://oeis.org/A007051>, and N(10)
should be 41. Let's try to follow the same algorithm, but this time I won't
draw it:

```
N(10) = N(8) + N(2) * N(6) + N(4) * N(4) + N(6) * N(2) + N(8)
N(10) = 14 + 1*5 + 2*2 + 5*1 + 14 = 42
```

Classic off-by-one, except it is not. If we now google the sequence
`1, 2, 5, 14, 42`, the first link is the wikipedia page for
[Catalan numbers](https://en.wikipedia.org/wiki/Catalan_number)
. These numbers seem to be very important in combinatronics, so this time
N(12) must be 132.

But we already have a nice algorithm, so let's implement it. We can easily add
grudges: before splitting the circle into two just check that the two people we
chose can form a pair.

```rust
struct Dance {
    p: u32,
    g: HashMap<u32, Vec<u32>>,
}
```

`p` is the number of people, and `g` is the list of grudges.
The grudges are offset-based: a grudge between 4 and 3 will be represented as
`g[3].push(4-3);`, so we can access all the grudges given the lowest index.
I used a `HashMap` instead of a `Vec` because I expected the grudges to be
sparse: not all people should have grudges.

This algorithm can be easily implemented in a nice recursive way: split the
dance into two until there's no more people left.

```rust
fn calc(&self) -> u32 {
    if self.p % 2 != 0 {
        return 0;
    }

    if self.p == 0 {
        return 1;
    }

    // recursion!
    let mut acc: u64 = 0;
    let mut i = 1;
    while i < self.p {
        // Num combinations = product of combinations of each part
        if let Some((dl, dr)) = self.split_into_two(0, i) {
            acc += (dl.calc() as u64) * (dr.calc() as u64);
            acc = acc % 1_000_000_007;
        }
        i += 2;
    }

    acc as u32
}
```

The `split_into_two` function returns `None` when there is a grudge between 0
and i, and otherwise returns the two new `Dance`s as `Some((dl, dr))`. These
new dances can then be recursively solved.

```rust
fn split_into_two(&self, i: u32, j: u32) -> Option<(Dance, Dance)> {
    if i != 0 {
        unimplemented!()
    }
    // check if 0 and j form a grudge
    if self.g.get(0).map_or(false, |x| x.contains(&j)) {
        return None;
    }

    let drp = j - i - 1;
    let mut drg = HashMap::new();
    // Change the offset of the grudges, since we remove the 0
    // Move g[x] to g[x-1]
    if drp > 0 {
        for x in i + 1..j {
            if let Some(grs) = self.g.get(&x) {
                for &gr in grs {
                    if gr < j - x {
                        drg.entry(x - 1).or_insert(vec![]).push(gr);
                    }
                }
            }
        }
    }
    let dr = Dance::new(drp, drg);

    // Same for dl
    // Move g[x] to g[x-j-1]
    // [...]

    Some((dl, dr))
```

And while this algorithm works pretty well for the test case, we can't use it
for the submit case because the recursion gets slower and slower with more
people. So while thinking about a solution, I added some optimizations.

For example, we can add a list of the Catalan numbers and use it when there
are no grudges, this should save a few levels of recursion:

```rust
// fn calc(&self) -> u32
let n = self.p as usize / 2;
const CATALAN: &[u32] = &[1, 1, 2, 5, 14, 42, 132, 429, 1430, 4862, 16796];
if n < CATALAN.len() && self.g.is_empty {
    return CATALAN[n];
}
```

Also, we can avoid checking the list of grudges in `split_into_two` and move
that check to `calc`, which will be much more efficient if we also sort the
vector of grudges.  But these optimizations are not enough, we need some way to
decrease the depth of the recursion, avoid calculating similar things more than
once, we need cache!

```rust
struct Cache {
    c: HashMap<Dance, u32>,
}
```

I created a dedicated cache struct to be able to control its behavior, we don't
want to run out of memory, which will happen if we keep all the entries. I was
planning to remove the "cheapest to calculate" (few people) first, but it
turned out to be unnecessary.

But we can't use the `Dance` as a key, because it's not hasheable. It contains
a `HashMap` of grudges, and that `HashMap` cannot be hashed (since its elements
are not ordered, two hashmaps with the same elements would return a different
hash, unless we sort the elements before hashing but that's very inefficient).
Luckly the `BTreeMap` can be hashed, and it provides a very similar interface
so we can replace one with the other without many changes to the code:

```rust
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
struct Dance {
    p: u32,
    g: BTreeMap<u32, Vec<u32>>,
}
```

Now add a few lines to the `calc` function:

```rust
fn calc(&self, cache: &mut Cache) -> u32 {
    // The cache is magic, it converts O(n!) into O(n^2)
    if let Some(x) = cache.get(self) {
        return x;
    }
    
    // [...]

    let acc = acc as u32;
    let clon = self.clone();
    cache.insert(clon, acc);

    acc
}
```

And the code is able to complete the submitInput! The runtime is pretty bad,
about 3 minutes, so there must be room for optimization. I don't know if the
algorithm is actually O(n^2), that's just something I read on stackoverflow
while searching for optimizations.

If I had to rewrite the problem I would store the grudges and the cache in one
struct, and make all the recursive partitions on other struct, this way we
avoid cloning and modifying the list of grudges when splitting, and also we can
use a `HashMap` to store the grudges, which is faster than a `BTreeMap`. Well,
let's just do that:

```rust
struct Dance {
    g: HashMap<u32, Vec<u32>>,
    cache: HashMap<DancePart, u32>,
}
impl Dance {
    fn calc(&mut self, p: u32) -> u32 {
        let x = DancePart { offset: 0, p: p };

        x.calc(&mut self)
    }
}

#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash)]
struct DancePart {
    offset: u32,
    p: u32,
}
impl DancePart {
    fn calc(&self, dance: &mut Dance) -> u32 {
    }
}
```
 
You can check out that alternative implementation:
[blog/p10\_hashmap](https://github.com/Badel2/TuentiChallenge8/blob/master/blog/p10_hashmap/p10_hashmap.rs)
, and the run time is now only 1.2 seconds, a 150x speedup!

I have also tried a version which uses a `Vec<Vec<u32>>` instead of a `HashMap`
to store the grudges, but the difference was very small, it was about 0.02s
faster.


# P11

### Diamonds

This was the last challenge I had time to begin, but I got stuck and wasn't able
to complete it.

The goal is to maximize the number of lasers with the restriction that two
lasers cannot cross on a diamond. A good starting point would be to set lasers
on all the rows, and leave the columns empty. Then, check if we can add lasers
to the columns without crossing any diamonds. So in the worst case, the
solution would be `max(num_rows, num_cols)`. But the best solution won't always
have a side full of lasers, sometimes we need to remove one laser to be able to
add two, like in this example:

```
   ABCD
     vv
1> **
2    **
3>  *
4> *
```

We remove the laser on row 2, and add two lasers on columns C and D.  I guess
we could check all the possibilities: remove one laser, try to find a stable
configuration, repeat with another laser. But the limits are N,M <= 500, so a
naive approach won't work.

Some trivial optimizations are removing empty rows and columns, which will
always have a laser, and splitting the problem into clusters when some groups
of diamonds are independent. In the example above, we can split the grid into
two:

```
**
 *
*
    **
```

And then it becomes clear that the solution is 3+2 = 5. It would be nice if all
problems could be easily solved using recursion, but this doesn't look like it.

So let's talk about the implementation. I decided to store the grid as a vector
of vectors of indices: `Vec<Vec<u32>>`, instead of storing it as a boolean
matrix like a normal person.

```
**
 *
*
```

In this example, the vector would look like this:

```
[
    [0, 1],
    [1],
    [0]
]
```

And there are a few useful helper functions: 
* `traspose`, which transposes the
matrix (switch rows and columns).
* `remove_empty_rows` removes the rows with
no diamonds, to remove the columns instead we just need to transpose and remove
rows.
* `roc` sorts the matrix so that the clusters can be trivially found later, using
the
[Rank Order Clustering](https://en.wikipedia.org/wiki/Production_flow_analysis#Rank_Order_Clustering)
algorithm, which is actually used to group machines by products in factories,
but hey it works here too.
* `find_clusters` takes the sorted matrix and divides it by the non overlapping
clusters of diamonds, which can be solved independently.

Here's an example of the output of the roc algorithm, can you see the 3
clusters?

```
**    
  **  
  * * 
   *  
    * 
     *
```

This simple clustering technique is almost enough when combined with
`max(num_rows, num_cols)` as the score, it only fails 8 cases from the 50 in
testInput. So I decided to analyze these cases, and I realized something very
important:

> Not all diamonds need a laser!

My algorithm would probably be the correct one if we need to secure all the
diamonds, but it is never said that we need a laser on all the diamonds. For
example:

```
   ABCD
    vvv
1  ****
2> *   
3> * 
4> *
```

Here if we leave the A1 diamond unprotected, we are able to put 6 lasers
instead of the trivial 4. 

And after taking that into account, I got stuck. The only clear idea I had was
bruteforce, but I thought that there must be some clever way. Maybe turn the
grid into a list of constrains: a diamond at A1 means that you either put a
laser on row 1, on column A, or nowhere. Maybe it would be enough to find the
largest rectangles with no diamonds inside the grid? Is there some way to solve
this using recursion?

It was the last day of the challenge but I still had about 9 hours to work on
this problem, and I managed to bruteforce the missing cases from testInput so I
could see the submitInput, and the clusters were about 80x80, so better forget
about bruteforce. I didn't want to skip this challenge, because the next one
had been skipped by many people, so it was probably one of these fun challenges
like the one with the gameboy cartridge, where you don't even know where to begin.
And I felt like I got pretty far in this challenge, and I was only missing one
small piece.

But in the end nothing happened.

# P12

### usb-ip

// TODO
It turns out that writing is pretty time consuming, so come back in a week or
so. I also plan on solving some more challenges, at least P12 which looks
pretty fun, but the later ones will probably stay unsolved for a while.

_Last updated 2018-05-25_
