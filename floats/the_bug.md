# The Hallucinated Rows Incident

Herein lies the tale of the serialization bug that caused one of the weirdest
crashes in the company's history- the infamous "Not enough values in db" panic.
Delve with me into the depths of implementing but a single SQL operator- `ORDER BY
... LIMIT`.

## The Big Diff With Our Engine

At Epsio Labs, we develop an incremental SQL engine- in brief, an SQL engine
that operates on changes in the data, instead of always using the whole dataset.
Explaining why that's useful I'll leave to our [documentation](https://docs.epsio.io/), but to
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
This list can be consolidated to a shorter one:
```
"rock"      : +1
"other_rock": +4
```
by adding together the modifications of the two different entries for `rock`.

An equally important concept is that of the default or null value- if we
consolidate together `"rock": +2"` and `"rock": -2`, we would get `"rock": 0`.
But of course, like we all learn in primary school, adding zero to a key is like
doing nothing at all- and so if the modification of a diff is zero, we simply
delete that diff.

Obviously, the typing for the modification of a diff is actually generic - so long
as the type fulfills these two properties, any valid type can be used, not only
signed integers[^1]. For the curious, In our rust-based engine, a modification type is represented using the following
trait constraint:
```rust
pub trait Modification: PartialEq + Default + Add<Output = Self>
```

[^1]: The mathematically astute among you might realize that formally a
    modification can be typed as any [commutative group](https://en.wikipedia.org/wiki/Abelian_group).

### Representing The Real World as Diffs

Imagine a person named Yumi. Yumi has the sacred task of stacking stones (don't
ask why, it's for good World Building reasons). In order to best stack stones, Yumi
gives the stones to a stone-stacking machine, which measures different parameters for each stone-
the weight, the volume, the shape- and Yumi later uses the machine to help pick the next best
stone to add to the stack.

For simplicity's sake, let's assume the machine only tracks the
weight of each stone- in such a case, the stacking machine might have a table
similar to this:

| stone_name   | weight |
|--------------|--------|
| big_rock     | 1.45   |
| medium_stone | 0.97   |
| small_pebble | 0.12   |

How would we represent these distinct lines as diffs? Well, easily enough-
Each row will be the key of the diff, with an integer modification
representing the amount of times the row appears in the table. For example, the above table
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
in the table with this value. After these two actions,
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

### Operations on Diffs

When data is represented as diffs, a query on the data can be thought of as
an operation on those diffs - this a complicated subject deserving a story of its
own, but at its core
what the Epsio engine does is represent SQL queries as a series of operations
on an incoming stream of diffs[^2].
A simple operation, like counting the number
of rows, can be easily represented as summing the modifications of diffs we got
as an input, and outputting a diff whose key is this result. For the previous
example, this counting operation would output `3: +1`, which translates to this
table:

| count |
|-------|
| 3     |

If we then give as input the diff `("big_rock", 1.45): -1` (i.e. deleting a row), our count operation would
output:
```
3: -1
2: +1
```
which can be read as "delete the row with value `3`, and add a row with value `2`".

Sorted that out? Great, now we can move to sorting.

[^2]: The exact reason why it is so vital to our Engine to represent data as
diffs which can be consolidated is beyond the scope of this tale, you can
simply assume it has to be this way.

## That's A Lot Of Stone To Sort Through

Right, so now Yumi's machine has an assortment of diffs, which represent all the
different stones Yumi can use for her stacks. What remains is choosing the right
stone to add to the top of the stack- but of course, there are a lot of stones to
consider, and so Yumi might decide to ease the load; she might choose to limit herself to
consider only the biggest 3 stones in the machine's database. In SQL, this query
is easy enough to write:
```sql
SELECT * FROM stones_table ORDER BY weight DESC LIMIT 3;
```
But how would we actually calculate it using diffs?

### The Stone Sorting Algorithm

<!--First, let's note a limitation of the diff model- a collection of diffs, unlike rows in a table,-->
<!--is inherently unsorted. Indeed, our engine (which Yumi's machine obviously-->
<!--uses), cannot output sorted tables, for a similar reason to why materialized views cannot do so.-->
<!--It still can, however, output the top 3 weightiest stones- even though they're-->
<!--unsorted.-->

<!--But under what model do we do any operation on diffs?-->

<!--In our engine, we use an input-state-output model. Each operation has a state,-->
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
To output the top 3 rocks, our engine has to first store all the rocks in some
sorted way. To do this, we of course picked [RocksDB](https://rocksdb.org/), an
embedded lexicographically sorted key-value store, which acts as the sorting
operation's persistent state. In our RocksDB state, the diffs are keyed by the value of
`weight`, and since RocksDB is sorted, our stored diffs are automatically sorted by their
weight.

Let's assume Yumi already inputted into the machine three diffs:
```
("huge_rock", 2.45): +1
("big_rock", 1.45): +1
("tiny_pebble", 0.13): +1
```
Since there are only 3 diffs, all three would be outputted, and of course all
inputted diffs are also written to RocksDB for persistent storage.

Now Yumi inputs another stone into the machine, and thus another diff into the
engine- for example, `("medium_stone", 0.97): +1`.
What we do (speaking as our royal engine) is query RocksDB for the top 3 diffs we previously stored.
The top 3 diffs in storage should be equal to our previous output, and are also all the data
that is relevant to calculate the new top 3 given the new diffs.

We add into this list the newly inputted diffs, and sort it in memory. Then we
can compare the sorted list of both new and previous diffs to the list of only
previous diffs- if the top 3 remained the same after introducing the new diffs,
then all inputted diffs are currently irrelevant- we store them in RocksDB in
case one of the top 3 is deleted, but we don't generate any output.

If, however, the top 3 diffs in the combined list are different to what's stored
in RocksDB, then we know the top 3 have changed- and we need to update our
output of this change: for each diff that
is no longer in the top 3, we output its negative, and for each new diff in the
top 3 we output its positive. Given the new diff Yumi inputted at the start of
this section (which was `("medium_stone", 0.97): +1`, in case you forgot);
and given the state as it was when she did so, our output would be
```
("tiny_pebble", 0.13): -1
("medium_stone", 0.97): +1
```
These two diffs are representing the change from `tiny_pebble` into `medium_stone` in
the table of top 3 stones by weight. Note here that if we consolidated all
diffs outputted by the sort operation, we should always remain with up to 3
positive diffs- the `tiny_pebble` negative diff cancels out the previously
outputted `tiny_pebble` positive diff, meaning the total count of diffs should
still be 3- after all, a "top 3" table with four rows in not exactly what Yumi
is looking for.

Here, you might already see a hint to the devastation that is about to befall Yumi's machine-
what happens when the sorting we do in-memory disagrees with the order RocksDB
sorts the same diffs?


## Floating Points and Broken storekeys

Before we dive right into that, let's expand on how _exactly_ are diffs stored.
Inherently, RocksDB supports storing only bytes- it has no concept of more
complicated objects, and so in practice serialization libraries are used to
convert complicated objects to and from their representation in RocksDB.

The engine uses [`rust_decimal::Decimal`](https://docs.rs/rust_decimal/latest/rust_decimal/) to
represent high precision decimal numbers, like the `weight` property.
Serialization of RocksDB keys is done by the [`storekey`](https://docs.rs/storekey/0.5.0/storekey/) crate.
To know how Yumi's machine stores diffs, we can now ask-

How does `storekey` serialize `rust_decimal`? <br/>
Well, using [evcxr](https://github.com/evcxr/evcxr) to run Rust in Jupyter,
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

This might seem completely nonsensical at first glance, but it actually makes a lot of sense-
the ASCII value of digits does correspond to their lexicographical order (0 is
0x30, 9 is 0x39); since the dot (0x2e) is smaller than all digits, 2.01 will sort
before 201; and since shorter numbers sort first, 2 will sort before both.
If we went ahead and used this default behaviour (which we
unintentionally did), for almost all stones, Yumi's machine would give completely
correct responses. No, the real problem is much less obvious and much more
_sinister_- and it ties into how decimal numbers represent their precision.

### Precision

When Yumi places a stone on a scale, the scale outputs a number- for example,
1.56. But as anyone who has tried to bake with _precisely_ 1 kilogram of flour
can attest to, the scale never rounds- if a stone weights exactly 2, Yumi's
scale will still write out 2.00. This is because the precision of the scale
is 2 digits after the dot, meaning it knows for a fact the stone doesn't weight
2.01- it might weigh 2.009 or 2.001, but the scale can guarantee those first
two digits after the dot are zero. Knowing the exact precision of a measurement or
calculation is often very valuable, which is why `rust_decimal` supports it-
only in this case, it proved quite fatal.

## The Bug

So, here's an interesting question- what happens when Yumi gives her machine a
stone which weights _exactly_ 0.970? Let's take look at how the input table is supposed
to look, with this weird `evil_stone`-

| stone_name   | weight |
|--------------|--------|
| huge_rock    | 2.45   |
| large_rock   | 1.97   |
| big_rock     | 1.45   |
| medium_stone | 0.97   |
| evil_stone   | 0.970  |

If everything worked correctly, this stone wouldn't matter- it isn't in
the top 3, it shouldn't affect the output at all. But, and this might surprise
you, not everything worked correctly- if we input this table in a random order, what we sometimes
get is in fact this:
```
Error while executing IVM 5c9e2a10-5d9a-4b41-8c29-63e3a076c24c:
    thread 'tokio-runtime-worker' panicked at 'Not enough values in db to pop 1 values (popped 0)'
```

[Huh?](https://youtu.be/hOLnLDEnUdU)

### The Issue

Let's run the algorithm by hand with the input. After Yumi surrendered four
random stones to the machine, we have this data in our RocksDB:

| stone_name   | weight |
|--------------|--------|
| huge_rock    | 2.45   |
| big_rock     | 1.45   |
| evil_stone   | 0.970  |
| medium_stone | 0.97   |

Alas, at this point Yumi's fate is already sealed- the fatal bug has been armed,
and is about to cause quite a panic. When Yumi added these four stones, our engine
should have outputted the top 3 stones- the top 2 are clearly `huge_rock` and
`big_rock`, but what does the engine output for the third? Well, the engine
sorts in memory, and when two decimals are equal in value-
even if one is more precise-`rust_decimal` considers them equal: `Decimal(0.97) == Decimal(0.970)`.

And so, when sorting two equal values, `Vec::sort` (which is what the engine uses) will essentially pick one at random-
add a little careful tracing, and bam! When it chooses `medium_stone` is
when the engine goes to live in a farm upstate.

This might still seem fine at first glance- indeed, if Yumi stopped inputting stones after these first 4 nothing would
seem broken- but a problem arises when a fifth new stone is added:<br/>
When `large_rock` is added, we query RocksDB for the top 3 values.
These should equal our output in the previous step - but didn't we let
`Vec::sort` pick the last stone basically at random? What does RocksDB pick?
Well, RocksDB is sorted lexicographically- longer values are larger, and so
`0.970` is always bigger than `0.97`. RocksDB has no fancy decimal libraries,
for it `0.970` is just a string. The third stone in RocksDB will _always_ be
`evil_stone`- meaning when `Vec::sort` picked `medium_stone` as the third stone
in the previous step, a desynchronization is created between our actual previous output and the
output we will calculate we sent.

### This Makes Even a Machine Panic

When we enter a new diff in this state (e.g. `("large_rock", 1.83): +1`), these are the top 3
results in RocksDB:

| stone_name   | weight |
|--------------|--------|
| huge_rock    | 2.45   |
| big_rock     | 1.45   |
| evil_stone   | 0.970  |

When we add this diff into the table and re-sort, the top 3 diffs change-
`evil_stone` is no longer in the top 3, replaced by `large_rock`, so we must
emit diffs that signify this change:
```
("large_rock", 1.83): +1
("evil_stone", 0.970): -1
```
But, during sorting, `Vec::sort` (which believes `0.97` and `0.970` are equal) picked `medium_stone` as the third diff,
which means that the `("evil_stone", 0.970): -1` diff is deleting the wrong entry- we _never outputted_ a positive `("evil_stone", 0.970): +1`,
we outputted `("medium_stone", 0.97): +1` instead- so what are we outputting this negative `evil_stone` diff for?

This is where the error message at the start of this tale rears its
unwelcome head- when this deletion of `evil_stone` arrives at the final output of the
engine, it has to be converted back into a regular row operation; in this case,
the deletion of a `("evil_stone", 0.970)` row from the engine's internal result
database. But of course, there is no such row- our internal DB can't pop a value
of which zero exist, and so it panics, like we saw before:
```
Error while executing IVM 5c9e2a10-5d9a-4b41-8c29-63e3a076c24c:
    thread 'tokio-runtime-worker' panicked at 'MACHINE! I know of no `evil_stone`! STOP asking me to delete it!
                                               Not enough values in db to pop 1 values (popped 0)'
```

Although I'm only paraphrasing, our internal results DB is actually very polite.


## Conclusion

So what did we learn?

If we take a step back, the reason this bug exists is that we sorted our input
in two different ways- once in-memory using `Vec::sort`, and once by serializing
our data and letting RocksDB sort it. We assumed these operations are equal,
but with the introduction of precision in decimal numbers it turns out they're
not.

So try to not do that, if you can- to put it dryly, Don't Repeat Yourself.<br/>
But in this specific case, the issue is that we _need_ to use both methods-
ditching RocksDB is obviously impossible, and using RocksDB to sort everything
makes our sort 3x slower (I know this because the previous sort algorithm did
just that). So what did we actually do? Well, that's a story for another time-
and another post.

### Wait, but what about Yumi, her stone-stacking machine, and its absolutely incredible engine?

I'm glad you asked. If you want to try Epsio's _blazingly fast_ (but
actually just cheating) SQL engine, [check us out here](https://www.epsio.io/).[^yumi]

[^yumi]: And if you want to know what happens to Yumi, I wholeheartedly
    recommend Brandon Sanderson's [Yumi and the Nightmare Painter](https://www.goodreads.com/book/show/60726999-yumi-and-the-nightmare-painter).
