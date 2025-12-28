## Extended NSV

STATUS: Early Draft

### Why?

The original drafts of NSV were inspired by what I was already using for embedding data inside of Markdown notes.  
That, among other things, meant that the format would have support for rich metadata, something that CSV is painfully lacking.  
But that is a far wider design space, one that would necessitate iteration over prolonged time, and thus versioning.  
So after a few not-so-successful attempts at making one, I decided to eschew any and all metadata from the core NSV format.  
That enabled the 'no versioning' stance for the core NSV, keeping the burden where it belongs, here.

### Not-yet

This is not yet a specification, but a collection of drafts on the matter.  
They have reached a point where there's a certain level of consistent vision, and that is what I am describing here.

The story begins with defining an NSV-aware operation on a seqseq.  
Terms are used according to their definition in the core spec.

NOTE: The `lift` section is now outdated, to be reformulated via `spill`, as defined in [properties](./properties.md).

#### Lift/unlift

Lift is an operation on a 'seqseq' that produces 'row lines' containing flattened data.  
So it takes a `Seq[Seq[String]]` and produces a `Seq[String]`, removing one dimension (without losing structural information!).

The rule is as follows
1. Apply escaping to every cell in every row
2. Chain the escaped cells, interleaving empty cells where row boundaries were

Unlift is the inverse, so
1. Walk element-by-element
2. If current element is non-empty, unescape it and append to the current row
3. If current element is empty, terminate the current row

There're multiple ways to think about it, some are
- in terms of array operations, this one is close to flatten/ravel, except we inject delimiters into the flattened sequence, instead of preserving shape data separately (can't do that with ragged rows without losing human editability)
  - do note that we do NOT encode 'rows as cells'
- we reinterpret the same data as a sequence of lines instead of parsing it as NSV, then encode it as a single row (not as a single-element seqseq! no 'empty cell' after the last element)

The 'effect' of applying lift is that we get to combine multiple rows into one in a reversable way.  
Why is that important? Because conventionally, the metadata is the first 'row' of the data.  
That means, we now have the options to both store metadata as a regular NSV (e.g. in a separate file), and as just a regular row (from core NSV parser's perspective) in the file itself.  

Everything in the ENSV that follows is but an 'interpretation' of data in a seqseq, with the metadata format not having a separate parser.  
In other words, ENSV is agnostic of the files themselves, and operates on the level of encoding to and from a `Seq[Seq[String]]`.

Ah, almost forgot, any ENSV is a valid NSV, can just skip the first row and get the data.  
A little more involved when metadata is interleaved, but that's a rare case, and also manageable.

#### Forms

Finally, something that is not newline this, newline that.  
The way I think about introducing extensible metadata is by defining a number 'forms', with rules like 'ignore unknown forms' for forward compatibility, and rules like 'rename when changing semantics' for backwards compatibility.  
This, of course, is inspired by Lisps. What I plan not to borrow from Lisps are parentheses, and here's the plan.  
In a way, NSV already represents a 'tree', except it is a 'truncated' tree with all leaves (cells) being at depth 2.  
As long as we keep metadata to nesting no deeper than 1–2 levels, we should be able to represent what parens would as NSV's structure itself.

So, forms shall fall into one of two categories
- cell forms, where both the form name and all of its arguments live within a cell (visually, a line)
- row forms, where the form name lives in the first cell (maybe with some of the arguments) and the rest of its arguments (arbitrary length!) lives as the rest of the row (visually, a paragraph)

The primary factor when choosing between the two is readability/editability.  
Long horizontal lines are not readable, and neither is text with severe vertical spread.  
The characters that would be escaped during lift (well, the backslash) are to be avoided (representable without issues, though, just ugly).

#### Features

These would largely correspond to forms, and can be seen as a list of such.  
I would not give them names yet, so as not to create confusion for later.

Cell forms
- a form to specify seqseq constraints
  - notably, one to indicate that the data forms a table
- a form to provide column names (horizontal)
- a form to provide column types (horizontal)
- a form to define sentinels for boolean type
- a form to define a sentinel for 'null', not sure how to handle the case where the empty string *is* the sentinel
- a form to indicate that in-cell comments are enabled and to provide the delimiter
- a form to specify the number of data rows that are to follow
  - useful for multiplexing in a stream, or to represent heterogenous data in a file (e.g. a list of nodes followed by a list of edges)
- a form to specify a version of the ENVS metadata itself (still unsure if I'd need this)

Row forms
- a form to provide column names (vertical)
- a form to provide column types (vertical)
- a form to provide column name-type pairs (vertical)
- a form to define rules for packing/unpacking multiple fields in one cells
  - useful e.g. for log layouts where wasting a line for each field is a damn waste

As for distinguishing between cell and row forms, since they're all defined a priori, there can be no ambiguity.  
For readability, a convention like "all row form named should end on `:`" would suffice.

---

That's it for now!

