## Properties

This section covers decomposition of NSV's encode/decode into more primitive operations and some operations the decomposition enables.  
Do note that implementations are unlikely to follow the strict boundaries between operations defined here due to performance costs doing so would entail.

### Types

Concrete
- `Char` — individual character/byte
- `String` = `Seq[Char]` — sequence of characters
- `Seq[String]` — sequence of strings (1D structure)
- `Seq[Seq[String]]` — seqseq (2D structure)

Abstract
- `Seq[A]` — sequence of elements of type `A`
- `E[A, t]` — type `A`, excluding element `t ∈ A`, think `A \ {t}`

Set of possible `Seq[E[A, t]]`s is obviously a proper subset of the set of possible `Seq[A]`s.  
Note that "exclusion" here does not refer to use of any particular escaping approach, what matters is that the resulting sequences no longer contain `t`.

### Operations

`spill[A, t]: Seq[Seq[E[A, t]]] → Seq[A]`  
The operation itself is simple: we "spill" structure into the flattened sequence, adding `t` every time a nested sequence ends.

`unspill[A, t]: Seq[A] → Seq[Seq[E[A, t]]]`  
Inverse of `spill`. Picks up the termination tokens from the sequence and uses them to recover the original structure.

`escape: String → E[Seq[E[Char, '\n']], '']`
This `escape` is, strictly speaking, a mapping rather than an escaping operation, as it must also map the empty string to a sentinel.  
Maps into a set of non-empty strings that cannot contain newline characters.  
The escaping rule as defined in the NSV spec is a special case of this operation, chosen for human readability and momentum.

`unescape`  
Left inverse of `escape`.

`encode: Seq[Seq[String]] → String`  
The entirety of NSV encode, take the input seqseq, produce the serialisation.

`decode: String → Seq[Seq[String]]`  
The inverse of `encode`.

### Decomposition

With definitions as above, `encode` is exactly  
`encode = spill[Char, '\n'] ∘ spill[String, ''] ∘ map(map(escape))`

`escape` is defined in a way that takes care of both layer's termination tokens in one go.  
Trivially, it was possible to use the same token for both levels, and repeat a simpler escaping schema, but I do not believe a readable variant exists among such encodings.

`decode = map(map(unescape)) ∘ unspill[String, ''] ∘ unspill[Char, '\n']`

Note that `unspill` does not depend on the escaping scheme — structure recovery precedes unescaping.  
Any mapping whose outputs are non-empty and LF-free is compatible.

### Repeated application

One problem that needed a solution was encoding NSV inside of an NSV.  
This problem arises at very least with the need to include metadata without inventing yet another encoding for structure.

The most straightforward way is of course using `encode` to force seqseq into a cell. This produces an unreadable mess, of course.  
But the nature of `spill`, collapsing just one dimension, enables two other escaping scenarios
- Use `spill[Char, '\n']` to collapse innermost dimensions, effectively converting every row into a cell
- Use `spill[String, '']` to collapse outermost dimensions, effectively converting the seqseq into a row

Both spills would require escaping the strings first.  
The former would have the unfortunate consequence of adding a lot of literal `\n`s, and bloating the result horizontally.  
The latter, though, produces a fairly readable result with every cell of the original seqseq being a cell in the final one.  
Since escaping is designed to be identity in all cases where it's not needed, a carefully designed metadata format may well preserve most of cells as-is, without paying extra readability/parsing costs for repeated escaping.

Both spills would produce a single row, that can then be included in the final seqseq as a regular element.  
In case of metadata, or any scenario where some of the rows are nested in such way, the information that the row must be treated specially would have to be sidecared.

### Paths not taken

There are alternative definitions that would not work in general case but offer interesting tradeoffs if something is known about the strings and sequences being encoded.

Most notable one is use of separators.  
NSV uses terminator semantics everywhere, including the spill operation definitions.  
This is necessary to make `[]` and `[[]]` distinguishable, which would not be possible if the structurally-significant token would only appear once sequence is of length ≥ 2.  
That said, in many real-world cases it is known that sequences/strings are non-empty.  
In those cases, omitting the final terminator would produce a more frugal encoding, which becomes especially noticeable once repeated application is used to encode multi-dimensional data (don't tho).

Another one, is not using escaping at all.  
Obvious enough, but some operations may produce strings that do not contain a given token.  
`unspill[Char, '\n']`, which pretty much just yields sequence of lines when given a file, is a notable example of that.  
If, by whatever means, you know that individual cells you are encoding can neither be empty nor contain LF characters, you may well not bother applying escaping at all. Saves the trouble of doubling every literal backslash.  
The only issue — you'd have to sidecar that information to the decoding step somehow.  
Core NSV does not support this approach whatsoever because it'd overly complicate handling real backslash sequences.  
And `escape` is already the identity when there's no `\`s, newlines, and the string is non-empty.

Yet another one, a generalisation of the previous.  
Since encode/decode cleanly decompose into a pair of spills/unspills plus escape/unescape, it is entirely possible to replace the escape/unescape step with anything, as long as it handles LF and the empty string.  
Then, technically, the pair of spills is defining what could be considered a family of encodings, with swappable escaping logic allowing to use whatever is least likely to have collisions with the data being encoded.  
For multidimensional ragged data, any escaping that does not do `\\` for escaping the escape character would do great. Sad we're in this world and not one where people know better than using escape character in its own escape sequence.  
For anything regex-heavy, or TeX, any escaping that doesn't make `\` special would do.

Minor, but the decomposition itself _could_ have been framed as `spill ∘ escape ∘ spill ∘ escape`.  
While cleaner from math perspective, it diverges too far from any implementation doing both "escapes" as one.  
It is still helpful when thinking about half-ready data, i.e. data types that are non-empty in all cases or are LF-free but not both.

