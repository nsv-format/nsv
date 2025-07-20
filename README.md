# Newline-separated values

STATUS: Draft

## Why? (advantages)

- Better Git diffs
- Simpler implementation
- More flexibility for annotations
- Better navigation in vim-like tools

See more in [pitches](./pitches.md).

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

Individual nested sequences are called "rows" and individual elements in them are called "cells" even when they do not form a table.

The general parsing rule for the entire file ("simple rule" hereafter) is as following
1. Split on consecutive newlines to get rows
2. Split each on single newline to get cells
3. Unescape special characters in each cell

### Encoding

Trivially, the encoding rule is exactly that but in reverse order, with operations replaced with their inverse.

#### Escaping characters in individual cells

Literal `\`s MUST be replaced with literal `\\`.
Newlines MUST be replaced with literal `\n`.
Fields that contain no value, and would normally be represented as an empty string, must be explicitly represented with a single backslash.
<!-- `\` would then correspond to an invalid string that would never be encoded by backslash-escaped encoding. -->
<!-- As to why the token is needed in the first place: we need it to make parsing unambiguous (new row vs empty field) while retaining seqseq representability. -->

#### Combining escaped cells into a seqseq

A single newline is added at the end of each cell to indicate its termination.
A single newline is then added at the end of each row to indicate its termination.

### Decoding

#### Recovering the structure

A newline immediately following another newline always indicates that a row was completed, even if said row was empty.
Two newlines would mean that a row has ended, three that an empty row followed after that.
A table would not contain zero-cell rows, so the rule would degenerate into splitting on double newlines.

#### Unescaping characters in individual cells

Interpret a single `\` as the empty string.
Going left-to-right, interpret `\\` as a single `\` and `\n` as the newline symbol.

##### Handling invalid cells

Readers SHOULD ignore escape sequences they do not recognise, passing them through with the literal backslash.
A dangling backslash at the end of a line SHOULD be stripped.
<!-- Between passing the dangling backslash through and stripping it, the latter is chosen because it makes the empty cell token a special case of this rule. -->

## Examples

### A trivial example

 col1 | col2 
:----:|:----:
  a   |  b   
  c   |  c   

```csv
col1,col2
a,b
c,d
```

```nsv
col1
col2

a
b

c
d
```

### A less trivial example

```nsv
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

This would roughly correspond to the following Markdown table (NSV is not a table though, and Markdown doesn't support tables without column names)

 0                                                                              | 1                                                        | 2                 
--------------------------------------------------------------------------------|----------------------------------------------------------|-------------------
 first                                                                          | row                                                      
 second                                                                         | row                                                      
 missing ->                                                                     |                                                          | <- missing        
 Roses are red<br>Violets are blue<br>This may be pain<br>But CSV would be, too | Tab\tseparated\tvalues<br>(would be left as-is normally) | Not a newline: \n 

## Extras

### On data types

The format itself is data type-agnostic.
Everything is a string, and interpretation of those strings is up to the reader.
<!-- For practical applications, parsers would normally tightly integrate with converters, but deciding on what strings mean what is not up to this spec with a sole exception: the empty field token, `\`. -->

### On metadata

The core NSV format does not have processing rules for metadata.
I am currently cooking up an extended version that does, under the constraint of it not interfering with the processing rule described above.
That is, the processing rule can be used without looking out for changes to it, ever.
If they were to happen, it'll be a different format with a different name and different set of libraries.

Here are some of the things that would likely be in the extended format
- naming columns in metadata
- specifying column types in metadata
- comment support in the data paragraphs themselves

### On parsers, version compatibility

The rule for encoding/decoding shall remain without changes, so there is no versioning whatsoever. <!-- Aka YOLO versioning -->
Implementation-wise, one of the primary goals of this design is to make it sufficiently simple to handle these files without a library in cases where performance is not critical.
I intend to provide a number of reference implementations across languages, which could be vendored since one shouldn't really worry about the logic going out of date.
Performance-conscious implementations would of course have to evolve together with underlying tooling, use what matches your case.
