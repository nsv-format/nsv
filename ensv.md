## Extended NSV

STATUS: Early Draft

### Why?

The original drafts of NSV were inspired by what I was already using for embedding data inside of Markdown notes.  
That, among other things, meant that the format would need support for rich metadata, something that CSV is painfully lacking.  
But that is a far wider design space, one that would necessitate iteration over prolonged time, and thus versioning.  
So after a few not-so-successful attempts at making one, I decided to eschew any and all metadata from the core NSV format.  
That enabled the 'no versioning' stance for the core NSV, keeping the burden where it belongs, here.

### Not-yet

This is not yet a specification, but a collection of thoughts on the matter.  
They have reached a point where there's a certain level of consistent vision, and that is what I am describing here.

The story begins with defining an NSV-aware operation on a seqseq.  
Terms are used according to their definition in the [core spec](./README.md) and the [properties](./properties.md).

#### Lift/unlift

Lift is an operation on a non-empty 'seqseq' that produces 'row lines' containing flattened data.  
It takes a `Seq[Seq[String]]` and produces a `Seq[String]`, removing one dimension (without losing structural information!).

The rule is as follows
1. Apply escaping to every cell in every row
2. Chain the escaped cells, interleaving empty cells where row boundaries were
  1. Note that contrary to core NSV, this implies separator semantics, with the post-lift row's boundary serving as the final terminator

Unlift is the inverse, so
1. Walk the `Seq[String]` element-by-element
2. If the current element is non-empty, unescape it and append to the current row
3. If the current element is empty, terminate the current row
4. At end of input, terminate the current row

In terms of `spill`, as defined in [properties](./properties.md),  
`lift = init ∘ spill[String, ''] ∘ map(map(escape))`  
where `init` discards the last element (guaranteed to be the empty string after `spill` application).

The reason for discarding the final terminator is that it makes lift/unlift operations line number-preserving.  
This property is useful for operations in IDEs/text editors that operate on multiple selected lines (think "comment selected lines").  
This comes at the cost of making `[]` irrepresentable, a sacrifice we're willing to make.  
Note that when applied to a line range, `lift` is equivalent to simply applying escape to every 'line' (the proof is left to the reader).

There're multiple ways to think about it, some are
- in terms of array operations, this one is close to flatten/ravel, except we inject delimiters into the flattened sequence, instead of preserving shape data separately (can't do that with ragged rows without losing human editability)
  - do note that we do NOT encode 'rows as cells'
- we reinterpret the same data as a sequence of unescaped 'lines' instead of parsing it as NSV, then encode it as a single 'row' (not as a single-element seqseq! no 'empty cell' after the last element)

The 'effect' of applying lift is that we get to combine multiple rows into one in a reversible way.  
Why is that important?
1. It allows us to encode seqseq as a seq, enabling more structural depth without introducing new rules
2. Conventionally, the metadata is the first 'row' of the data. Lift makes it possible to "hide" all metadata as one row, which lets pure NSV readers discard all of it at once.

#### Extending spill/unspill

As per properties, NSV's encode is merely a composition of two spills applied to already-escaped data with different terminator tokens.  
Metadata is shaped as a seqseq, with the empty row not seeing any meaningful use.
This means `spill[Seq[String], EmptySeq]` is fair game.  
The escaping step is not necessary since metadata's full set of possible rows is spec-controlled, so if a certain row shape (empty, in this case) is designated as the end-of-metadata, the rest of the seqseq is guaranteed to not contain that.

That means, we now have the option to both store metadata as a regular NSV (e.g. in a separate file), and as just a prefix of rows (from core NSV parser's perspective) in the file itself.  
This should be fairly convenient if there's need to split data itself. One can imagine a directory with one NSV containing metadata, and the rest being large chunks of NSV data to which said metadata applies.  
To make any chunk a self-describing ENSV, you just concatenate the metadata, an empty row (literally one LF char), and the data.

A little more involved when metadata is interleaved, but that's a more special case, and also manageable.

#### Just a seqseq

Everything in the ENSV that follows is but an 'interpretation' of data in a seqseq, with the metadata format not having a separate parser.  
In other words, ENSV is agnostic of the files themselves, and operates on the level of encoding to and from a `Seq[Seq[String]]` (but that prior to applying unescape).  
It may, of course, further tokenize/parse the individual `String`s of said seqseq.

Any ENSV is a structurally valid NSV, i.e. row and cell boundaries are always recoverable regardless of the escaping rule in use.  
Note that the metadata parsing always uses the NSV's escaping rule.
The data can sometimes benefit from no-op escaping, or other changes to the rule.

#### Forms

Finally, something that is not newline this, newline that.  
The way I think about introducing extensible metadata is by defining a number 'forms', with rules like 'ignore unknown forms' for forward compatibility, and rules like 'rename when changing semantics' for backwards compatibility. <!-- I know compat is not that simple 😭 -->  
This, of course, is inspired by Lisps. What I plan not to borrow from Lisps are parentheses, and here's the plan.  
In a way, NSV already represents a 'tree', except it is a 'truncated' tree with all leaves (cells) being at depth 2.  
As long as we keep metadata to nesting no deeper than 1–2 levels, we should be able to represent what parens would as NSV's structure itself.

Each form would correspond to a single row (visually, a paragraph).  
The form name (maybe with some of the arguments) would live in the first cell.  
The rest of its arguments (arbitrary length!) would live as the rest of the row.

The primary factor in the form design is readability/editability.  
Long horizontal lines are not readable, and neither is text with severe vertical spread.  

#### Features

These would largely correspond to forms, and can be seen as a list of such.  
I would not give them names yet, so as not to create confusion for later.

Forms
- a form to group global constants (see below)
- two competing forms for field description
    - a form to provide column names–type pairs (same line)
    - a form to provide names, types, and additional metadata for each field
- a form to define rules for packing/unpacking multiple fields in one cell
  - useful e.g. for log layouts where wasting a line for each field is a damn waste
- a form to describe variants (discriminated sums)

Constants
- whether the seqseq is a table
- sentinels for boolean type
- sentinel for 'null'
- whether in-cell comments are enabled + the delimiter
- data section end sentinel (choice out of a fixed set)
  - useful for multiplexing in a stream, or to represent heterogenous data in a file (e.g. a list of nodes followed by a list of edges)
- the style of metadata update (only used with the above one)
- the version of the ENSV metadata itself (still unsure if I'd need this)

A 'column' is the set of all 'cells' at a given offset across all 'rows'.  
That is, column names and types should be available for generic seqseqs just as they are for the 'tables'.
I use 'field' and 'column' largely interchangeably, with the former being intended for a more generic case.  
The nuance would only matter once variant encoding is settled, which is not the case yet.

---

That's it for now!

