
# The Hallucinated Rows Incident

Herein lies the tale of the serialization bug that caused one of the weirdest
crashes in the company's history- the infamous "Not enough values in db" panic.

## What is Epsio + Story

- incremental DB engine
- Yumi example

### The Big Diff

At Epsio Labs, we develop an incremental SQL engine. Explaining why that's
useful I'll leave to our [documentation](https://docs.epsio.io/), but to
understand this bug it is important you understand a little how our engine
works.

The Epsio Engine represents data as a `Diff`- each `Diff` is a key-value pair,
specifying a modification to the value of some specific key. Internally, we
usually write a diff like so: `"rock": +1`. This diff can be read as "add
one to the value of the string `rock`". The important property of a diff
is that diffs on equal keys can be consolidated together into one diff by adding
their modifications together- consider for example this list of diffs:
```
"rock"      : +3
"other_rock": +4
"rock"      : -2
```
This list can be consolidated a shorter one:
```
"rock"      : +1
"other_rock": +4
```
by adding together the modifications of the two different entries for `rock`.

Another important concept is that of the default or null value- if we
consolidate together `"rock": +2"` and `"rock": -2`, we would get `"rock": 0`,
but of course a modification by zero makes no sense- and the obvious solution is
to simply drop the diff.

Of course, the typing for the modification of a diff is generic- so long
as the type fulfills these two properties, any valid type can be used, not only
signed integers[^1]. In rust, a modification is represented using the following
trait constraint:
```
pub trait Modification: PartialEq + Default + Add<Output = Self>
```

When data is represented as diffs, a query on the data can be thought of as
an operation on those diffs- this a complicated subject deserving a story of its
own, but at its core
what the Epsio engine does is represent SQL queries as a series of operations
on an incoming stream of diffs[^2].


[^1]: The mathematically astute among you might realize that formally a
    modification can be typed as any [commutative group](https://en.wikipedia.org/wiki/Abelian_group).

[^2]: The exact reason why it is so vital to our Engine to represent data as
diffs which can be consolidated is beyond the scope of this tale, but for now
simply assume it has to be this way.


### Representing The Real World as Diffs

Imagine a person named Yumi. Yumi has the sacred task of stacking stones (don't
ask why, it's good world building). In order to best stack stones, Yumi
estimates different parameters for each stone- the weight, the volume, the shape- and inputs
those parameters to a stone-stacking machine, which uses them to always pick the next best
stone to add to the stack. For simplicity's sake, lets assume the machine only tracks the
weight of each stone- in such a case, the stacking machine might have a table
similar to this:

| stone_name   | weight |
|--------------|--------|
| big_rock     | 1.45   |
| medium_stone | 0.97   |
| small_pebble | 0.12   |

How would we represent these distinct lines as diffs? Well, easily enough-
Let's decide that each row is the key of the diff, with the integer modification
representing the amount of times the diff appears- meaning the above table
translates to:
```
("big_rock", 1.45)    : +1
("medium_stone", 0.97): +1
("small_pebble", 0.12): +1
```

If Yumi loses `small_pebble`, she can simply give the machine the diff
`("small_pebble", 0.12): -1`, which would cancel out the previous +1
to the key of `("small_pebble", 0.12)`, deleting the key since its value is 0.
If Yumi finds another `medium_stone` which weighs `0.97`, she can use the diff
`("medium_stone", 0.97): +1` to mark the addition of another stone with those
properties, resulting in the diff `("medium_stone", 0.97): +2` which represents the fact that there are two rows
in the table with the values `("medium_stone", 0.97)`. After these two actions,
Yumi's machine will have these diffs in total:
```
("big_rock", 1.45)    : +1
("medium_stone", 0.97): +2
```
Which would represent this table:

| stone_name   | weight |
|--------------|--------|
| big_rock     | 1.45   |
| medium_stone | 0.97   |
| medium_stone | 0.97   |

## How Do We Sort

- What is sorting
- The Algo
- Rocksdb and sortkey
- Rock Weight Example

Right, so now Yumi's machine has an assortment of diffs which represent all the
different stones it can use for its stacks. What remains is choosing the right
stone to add to the top of the stack- but of course, there are a lot of stones to
consider, and so Yumi might decide ease the load on the machine and limit it to
consider only the biggest 3 stones in its database. In a SQL, this is easy
enough to write:
```sql
SELECT * FROM stones_table ORDER BY weight DESC LIMIT 3;
```
But how would we it using diffs?

### The Stone Sorting Algorithm

First, let's note a limitation of the diff model- a collection of diffs, unlike rows in a table,
is inherently unsorted. Indeed, our engine (which Yumi's machine obviously
uses), cannot output sorted tables, for a similar reason to why materialized views cannot do so.
It still can, however, output the top 5 weightiest stones- even though they're
unsorted.

<!--But under what model do we do any operation on diffs?-->

<!--In our engine, we use a input-state-output model. Each operation has a state,-->
<!--which some sort of embedded DB, which it uses to store diffs relevant to its-->
<!--output. When new diffs are given as input, the operation uses them and the diffs-->
<!--stored in its state to output some new diffs and updates its state.-->
Let's imagine the wanted result- given this input data:

| stone_name   | weight |
|--------------|--------|
| huge_rock    | 2.45   |
| big_rock     | 1.45   |
| medium_stone | 0.97   |
| small_stone  | 0.77   |
| tiny_pebble  | 0.13   |

We want to output these diffs:
```
("huge_rock", 2.45): +1
("big_rock", 1.45): +1
("medium_stone", 0.97): +1
```
To output the top 3 stones, our engine has to first store all the stones in some
sorted way- to do this, we of course picked [RocksDB](https://rocksdb.org/), an
embedded lexicographically sorted key-value store, which acts as the sorting
operation's state. In our RocksDB state, the diffs are indexed by the value of
`weight`, and since RocksDB is sorted, our stored diffs are sorted by their
weight.
Lets assume Yumi already inputted into the machine `("huge_rock", 2.45): +1`, `("big_rock", 1.45): +1` and
`("tiny_pebble", 0.13): +1`. Since there are only 3 diffs, all three would
be outputted.

When new diffs are inputted into the sorting operation- let's say `("medium_stone", 0.97): +1`,
we query RocksDB for the top 3 diffs we previously stored. The top 3 diffs
currently stored should be equal to our previous output, and also all the data
that is relevant to calculate the new top 3 given the new diffs.

We add into this list the newly inputted diffs, and sort it in memory. Then we
can compare the sorted list of both new and previous diffs to the list of only
previous diffs- if the top 3 remained the same after introducing the new diffs,
then we only need to update the state with the new diffs- no output is required.

If, however, the top 3 diffs in the combined list is different to what's stored
in RocksDB, then we need to update our output of this change- for each diff that 
is no longer in the top 3, we output its negative, and for each new diff in the
top 3 we output its positive. Given the new diff `("medium_stone", 0.97): +1`,
and state as I previously described, our output would be
```
("tiny_pebble", 0.13): -1
("medium_stone", 0.97): +1
```
representing the change that `tiny_pebble` was replaced with `medium_stone` in
the table of top 3 stones by weight. Note here that if we consolidated all
diffs outputted by the sort operation, we should always remain with up to 3
positive diffs- the `tiny_pebble` negative diff should have canceled out the previously
outputted positive `tiny_pebble` diff, meaning the total count of diffs should
still be 3.

Here, you might already see where this is going- what happens when the diffs we
actually previously outputted doesn't match the top 3 values currently stored in
RocksDB?


## How are FP numbers stored

- `storekey` + `rust_decimal`
- what is precision
- how `rust_decimal` deals with precision

Before we dive right into that, let's expand on how _exactly_ are diffs stored.
Inherently, RocksDB supports storing only bytes- it has no concept of more
complicated objects, and so in practice serialization libraries are used to
convert complicated objects to and from their representation in RocksDB.
Yumi's machine needs to sort its diffs by the `weight` property, which is
decimal, and so our engine would use [`rust_decimal::Decimal`]() as the base object,
and [`storekey`]() in order to create a lexicographically sortable serialization
of the `rust_decimal::Decimal` object.
<!--Change here to make more accidental-->

How does `storekey` serialize `rust_decimal`? Well, using evcxr to run Rust in Jupyter,
the answer is as a null-terminated string:
```rust
use std::str::{FromStr, from_utf8};
use rust_decimal::Decimal;
use storekey;

let a = Decimal::from_str("3.141").unwrap();
from_utf8(&storekey::serialize(&a).unwrap()).unwrap()
```
```
Output[1]: "3.141\0"
```

This might seem completely nonsensical at first glance, but it actually makes some sense-
the ASCII value of digits does correspond to their lexicographical order (0 is
0x30, 9 is 0x39); and since the dot (0x2e) is smaller than all digits, 2.01 will sort
before 201; and since shorter numbers sort first, 2 will sort before both.
If we went ahead and used this default behaviour (which we
accidentally did), for almost all stones, Yumi's machine would get completely
correct responses. No, the problem is actually much less obvious and much more
sinister- and it lies in how decimal numbers represent their precision.

### Precision

<!--Floating point numbers are famously [hard]() for programmers to deal with. -->
When Yumi places a stone on a scale, the scale outputs a number- for example,
1.56. But as anyone who has tried to bake with _precisely_ 1 kilogram of flour
can attest to, the scale never rounds- if a stone weights exactly 2, Yumi's
scale will still write out 2.00. This is because the precision of the scale
is 2 digits after the dot, meaning it knows for a fact the stone doesn't weight
2.01- it might weigh 2.009 or 2.001, but the scale can guarantee those first
two digits after the dot are zero. Knowing the exact precision of a measurement or
calculation is often very valuable, which is why `rust_decimal` supports it-
only in this case, it proved fatal.

## The Bug

- The question
- What the algo does
- produced diffs
- What happens

So, here's an interesting question- what happens when Yumi gives her machine a
stone which weights exactly 0.900? Let's take look at how the input table is suppose
to look-

| stone_name   | weight |
|--------------|--------|
| huge_rock    | 2.45   |
| big_rock     | 1.45   |
| medium_stone | 0.97   |
| evil_rock    | 0.900  |
| small_stone  | 0.77   |
| tiny_pebble  | 0.13   |

If everything would've worked correctly, this stone wouldn't matter- it isn't in
the top 3, it shouldn't effect the output at all. But, in fact, not everything
worked correctly- if we input this table in a random order, what we sometimes
get is in fact this:
```
Error while executing IVM 5c9e2a10-5d9a-4b41-8c29-63e3a076c24c:
    thread 'tokio-runtime-worker' panicked at 'Not enough values in db to pop 1 values (popped 0)'
```

[Huh?](meme)

### The Issue

Let's run the algorithm by hand with the input. In RocksDB, we have this data:

## Conclusion
