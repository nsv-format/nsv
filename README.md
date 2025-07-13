# Newline-separated values

STATUS: Draft, pre-v1

## Why? (advantages)

- Better Git diffs
- More flexibility for annotations
- Simpler implementation
- Better navigation in vim-like tools

See more [pitches](./pitches.md).

## Why? (history)

As I was ramping up yet another pet project, I wondered: how should I store the seed data for my project?
A shared dataset for local and prod, and maybe more envs. One that needs to be in sync on all of them. One that should be easily editable without special tools so that I could iterate fast. One that could be versioned without implementing yet another change tracking scheme.

And it sucked.
Binary formats and databases are not editable without special tooling. Versioning, where supported, is also not trivial.
Text formats, well, they all kinda suck?
JSON, even if JSON5 is bloat, with extra pain points if you're encoding a table.
CSV fails at both ends of making versioning visible: not only a change in one field results in diff for the entire line, it actually also doesn't for arbitrary text data in fields.
YAML is, well, YAML, just ask a Norwegian.

Enter NSV.
As naming should suggest, the main idea is to do what a CSV does, but separate fields with a newline.
Rows are then separated with a pair of newlines (at this point we have to introduce the empty field token).
I thought of it as a funny idea that probably had lots of issues, otherwise why wouldn't it be around.
But Claude said he knows no encoding format similar to what I was describing, and that brings us here.
The more we iterated on it, the more it made sense, and the less made its absence.
So here I am, bringing yet another encoding format into the accursed development world of today.

## Is this to CSV what CSV is to TSV?

Here, naming may be somewhat confusing, but no.
Iterating on it a bit made me realise that I'd have to reimplement all the common tooling, and at that point, why would I bring with me all the design baggage of those formats?

So first, NSV is not a "table" format.
What is being encoded is a "sequence of sequences" (I'll refer to them as *seqseq*s).
*Exactly one* layer of nesting, and if all the nested ones happen to have same length — that's a table!
(I'm fighting off the desire to call the nested ones "subsequences" but the word exists, and so shall my suffering.)

Second, the handling of special characters (only the newline in this case) is different (no quoted bs).
Making something-SV formats deal with raw text was a horrible decision, and people duly abused it as one would expect.
This one shall not make that mistake.

Lastly, I intend to allow for somewhat richer metadata.

See even more [yapping](./yapping.md).

## Specification

If you find any part of this ambiguous, please do not hesitate to suggest improvements.

An NSV file has two parts with distinct processing rules
1. Header, containing metadata
2. Body, containing the data itself

The header is *not optional* and ends at the first `---` line (exact match) encountered in file.
<!-- One may recognise this as being heavily reminiscent of Markdown frontmatter, except for lacking opening horizontal rule. -->
<!-- Since we're starting fresh, there's no requirement to be able to parse data that may not contain the header, and we'd want at least the version to be there for future compatibility. -->

### Header

Each line in the header is interpreted as a literal string.
Implementations MUST preserve the order of lines in the header when they write a previously read file.
<!-- This weird one is a consideration for comments. -->

Lines that match a known pattern MUST be interpreted according to its processing rule.
<!-- Writers SHOULD attempt to place the version label before other known labels, so as to aid the parser. -->

Lines that start with `//`, `#`, `(`, `[`, `{`, or `-- ` **do not and never will** have any special meaning under this specification.
As such, you can use those to comment or keep any amount of metadata.
Lines that start with `x-` will also be ignored as a consideration for extensions.

 Pattern     | Interpretation          | Example | Note                   
-------------|-------------------------|---------|------------------------
 `v<number>` | Version number          | `v1`    | Assume `v1` if omitted 
 `table`     | Flag indicating a table | `table` |

`table` processing rule
1. Infer the number of columns from the first row
2. Discard rows that have any other number of cells; WARN by default

<!-- `table:<number>` was supposed to indicate that the file is representing a table of `<number>` columns; if a row has a mismatched number of cells, it is to be considered invalid. -->

### Body

Individual nested sequences are called "rows" and individual elements in them are called "cells" even when they do not form a table.

 col1 | col2 
------|------
 r1c1 | r1c2 
 r2c1 | r2c2 

```csv
r1c1,r1c2
r2c1,r2c2
```

```nsv
r1c1
r1c2

r2c1
r2c2
```

The general parsing rule for the body ("simple rule" hereafter) is as following
1. Split on consecutive newlines to get rows
2. Split each on single newline to get cells
3. Escape special characters in each cell

#### 1. Split on consecutive newlines to get rows

A newline immediately following another newline always indicates that a row was completed, even if said row was empty.
Two newlines would mean that a row has ended, three that an empty row followed after that.
A table would not contain zero-cell rows, so the rule would degenerate into splitting on double newlines.

#### 2. Split each on single newline to get cells

Nothing to add.

#### 3. Escape special characters in each cell

Fields that contain no value, and would normally be represented as an empty string, must be explicitly represented with a single backslash.
Literal `\`s MUST be replaced with literal `\\`.
Newlines MUST be replaced with literal `\n`.
<!-- `\` would then correspond to an invalid string that would never be encoded by backslash-escaped encoding. -->
<!-- As to why the token is needed in the first place: we need it to make parsing unambiguous while retaining the simplicity of implementation. -->

Writers MAY escape additional special characters, as long as it does not interfere with `\n` and `\\`.
Readers SHOULD ignore escape sequences they do not recognise, passing them through with the literal backslash.

#### Data types

The format itself is data type-agnostic.
Everything is a string, and interpretation of those strings is up to the reader.
<!-- For practical applications, parsers would normally tightly integrate with converters, but deciding on what strings mean what is not up to this spec with a sole exception: the empty field token, `\`. -->

#### An example

```nsv
v1
# Yappy yappy yap
---
first
row

second
row

missing ->
\
<- missing

Roses are red\nViolets are blue\nThis may be pain\nBut CSV would be, too
Tab\tseparated\tvalues\n(would be left as-is normally)
Not a newline: \\n
```

This would roughly correspond to the following Markdown table (NSV is not a table though)

 0                                                                              | 1                                                        | 2                 
--------------------------------------------------------------------------------|----------------------------------------------------------|-------------------
 first                                                                          | row                                                      
 second                                                                         | row                                                      
 missing ->                                                                     |                                                          | <- missing        
 Roses are red<br>Violets are blue<br>This may be pain<br>But CSV would be, too | Tab\tseparated\tvalues<br>(would be left as-is normally) | Not a newline: \n 

### Possible future extensions

Likely
- naming columns in metadata
- specifying column types in metadata

Under consideration
- comment support in the data paragraphs themselves

### Version compatibility considerations

At this point, there is assumed to be only one version `v1`, assumed by default.
Further extensions would be introduced by adding special flags in the metadata, so, hopefully, one would be enough.

For an implementation to "support" a given spec version, it must
1. **If** it interprets any optional labels, it must match the semantics described in spec exactly
2. Provide a way to view the full list of optional features, with indication of which ones are implemented (in whichever way is appropriate for the technology)
