<a href="https://creativecommons.org/licenses/by-nd/4.0/"><img src="https://licensebuttons.net/l/by-nd/4.0/88x31.png" height="16" alt="CC BY-ND 4.0"></a> naming · <a href="https://nsv-format.org">nsv-format.org</a>

# Newline-separated values

Implementations/installation instuctions are at the bottom of this file.

## Why? (advantages)

- Better Git diffs
- Simpler implementation
- Better navigation in vim-like tools
- Better encoding/decoding performance

See more in [pitches](./pitches.md).

## Why? (history)

As I was ramping up yet another pet project, I wondered: how should I store the seed data?  
A shared dataset for local and prod, and maybe more envs. One that needs to be in sync on all of them. One that should be easily editable without special tools so that I could iterate fast. One that could be versioned without implementing yet another change tracking scheme.

And it sucked.  
Binary formats and databases are not editable without special tooling. Versioning, where supported, is also not trivial.  
Text formats, well, they all kinda suck?  
JSON, even if JSON5, is bloat, with extra pain points if you're encoding a table.  
CSV fails at both ends of making versioning visible: not only a change in one cell results in diff for the entire line, it also *doesn't* for arbitrary text data in cells (where quotation is involved).  
YAML is, well, YAML, just ask a Norwegian.

Enter NSV.  
As naming should suggest, the main idea is to do what a CSV does, but separate cells with a newline.  
Rows are then separated with an additional newline (at this point we have to introduce the empty cell token).  
I thought of it as a funny idea that probably had lots of issues, otherwise why wouldn't it be around.  
But Claude said he knows no encoding format similar to what I was describing, and that brings us here.  
The more we iterated on it, the more it made sense, and the less made its absence.  
So here I am, bringing yet another encoding format into the accursed development world of today.

## Is this to CSV what CSV is to TSV?

Here, naming may be somewhat confusing, but no.  
Iterating on it a bit made me realise that I'd have to reimplement all of the common tooling, and at that point, why would I bring with me all of the design baggage of those formats?

So first, NSV is not a "table" format.  
What is being encoded is a "sequence of sequences" (I'll refer to them as *seqseq*s).  
*Exactly one* layer of nesting, and if all the nested sequences happen to have the same length — that's a table!  
(I'm fending off the desire to call the nested ones "subsequences" but the word exists, and so shall my suffering.)

Second, the handling of special characters (only the newline in this case) is different (no quoted bs).  
Making something-SV table formats deal with arbitrary text data was a horrible decision, and people duly abused it as one would expect.  
This one shall not make that mistake.

See even more [yapping](./yapping.md).

## Specification

Terms
- 'row' is an individual nested sequence
- 'cell' is an individual element of a 'row'
- 'line' is a sequence of characters between newlines
  - 'row lines' are the sequence of lines matching a row
- 'table' is a seqseq such that the length of all of its 'rows' is the same
  - 'row' and 'cell' are terms used even when the seqseq is not a 'table'
- 'newline' is the LF character
  - CR is not recognised as a special character in any scenario

NSV encodes a seqseq.  
A seqseq is a sequence of rows.  
A row is a sequence of cells (not lines!).  
A table is but a special case of seqseq and has no parsing rules beyond those of the general case.

Newlines are 'line' boundaries.  
Empty 'lines' are 'row lines' boundaries.  
Non-empty 'lines' are escaped 'cell' content.

The general parsing rule ("simple rule" hereafter) is as following
1. Accumulate characters into a buffer until newline
2. If the buffer is empty: complete the current row
3. If the buffer is non-empty: unescape buffer, add to current row as cell
4. Clear the buffer and continue

### Encoding

Trivially, the encoding rule is exactly that but in reverse order, with operations replaced with their inverse.

#### Escaping characters in a cell

Literal `\`s MUST be replaced with literal `\\`.  
Newlines MUST be replaced with literal `\n`.  
Empty cells MUST be explicitly represented with a single `\`.  
<!-- `\` would then correspond to an invalid string that would never be encoded by backslash-escaped encoding. -->
<!-- As to why the token is needed in the first place: we need it to make parsing unambiguous (new row vs empty cell) while retaining seqseq representability. -->

#### Combining escaped cells into a seqseq

A single newline is added at the end of each cell to indicate its termination.  
A single newline is then added at the end of each row to indicate its termination.

### Decoding

#### Recovering the structure

A newline immediately following another newline always indicates that a row was completed, even if said row was empty.  
Two newlines would mean that a row has ended, three that an empty row followed after that.  

#### Unescaping characters in individual cells

Interpret a single `\` as the empty string.  
Going left-to-right, interpret `\\` as a single `\` and `\n` as the newline.

#### Handling rejected strings

The following MAY/SHOULD rules are for strings that could not have been produced by a valid NSV writer.  
It is up to the user-facing tool to decide whether to coerce, warn, or error upon encountering an invalid string.  
Below are the coercion rules for all classes of rejected strings.

##### Handling invalid cells

Readers SHOULD ignore escape sequences they do not recognise, passing them through with the literal backslash.  
A dangling backslash at the end of a line SHOULD be stripped.  
<!-- Between passing the dangling backslash through and stripping it, the latter is chosen because it makes the empty cell token a special case of this rule. -->

##### Handling abrupt EOF

Any seqseq but [] and [[]] would end in two newlines once encoded.  
Readers MAY emit the incomplete cell/row.
<!-- Can't really prohibit this since human-edited files would often have this issue. -->
<!-- Can't really accept it as valid since it would get in the way of streaming/tailing. -->

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
<!-- For practical applications, parsers would normally tightly integrate with converters, but deciding on which strings mean what is not up to this spec. -->

### On metadata

The core NSV format does not have processing rules for metadata.  
I am currently cooking up an [extended version](./ensv.md) that does, under the constraint of it not interfering with the processing rule described above.  
That is, the processing rule here can be used without looking out for changes to it, ever.  
If they were to happen, it'd be a different format with a different name and different set of libraries.

### On parsers, version compatibility

The rule for encoding/decoding shall remain without changes, so there is no versioning whatsoever. <!-- Aka YOLO versioning -->  
Implementation-wise, one of the primary goals of this design is to make it sufficiently simple to handle these files without a library in cases where performance is not critical.  
I intend to provide a number of reference implementations across languages, which could be vendored since one shouldn't really worry about the logic going out of date.  
Performance-conscious implementations would of course have to evolve together with underlying tooling, use what matches your case.

### Specification, implementation, tool

The mental model I have for this is that these are the three distinct layers with distinct responsibilities.  
This repo, the specification, defines how to encode and recover the 2D structure. A simple task, a simple spec.  
An implementation, which may well be called a parser, stays true to the spec and has no responsibilities beyond that.  
A tool, then, is whatever is interfacing with the end user, be it a standalone CLI tool, an extension or plugin for some other tool, or a public method of the package holding the parser.

While the line between the parser and the tool may be blurred in code, the layered model itself is meant to address some of the spots not addressed by the specification itself.  
For one, dealing with real-world issues like files corrupted with CRs or BOM is in no way a responsibility of the specification. I understand that for performance reasons some tools may choose to embed the handling directly into the parser, but the burden of reconciling that with the format's layout should lie with such tools, and not the specification itself.  
Another case would be providing good ergonomics. Resumable (e.g. streaming, tailing) and non-resumable (e.g. file, in-memory string) consumption would require different handling of edge cases to provide good user experience. For example, in non-resumable case the EOF guarantees the end of data was reached, enabling more forgiving treatment of missing newlines at the end.

### On encoding

Since LF, `\`, and `n` are the only anyhow special bytes in the format, any encoding using the same bytes to represent these as ASCII, and does not use these bytes anywhere else, would work with NSV.  
This includes UTF-8, of course, but is very much not limited to it.

## Implementations

All implementations are licensed under MIT.

Repository | Language | Installation | Notes
--- | --- | --- | ---
[nsv-format/nsv-python](https://github.com/nsv-format/nsv-python) | Python | `pip install nsv` | Can patch itself onto Pandas (no type narrowing though)
[nsv-format/nsv-scala](https://github.com/nsv-format/nsv-scala) | Scala | Not published | String-only, the algo is calque from Python version
[nsv-format/nsv-rust](https://github.com/nsv-format/nsv-rust) | Rust | `cargo add nsv` | Reasonably fast
[nsv-format/nsv-js](https://github.com/nsv-format/nsv-js) | JS | `npm install @nsv-format/nsv` | Supports streaming

If you have ideas on how to make any of these more ergonomic or feel like implementing a brutally optimised version, I very much welcome that.
<!-- Of mine, I mostly used the Python and the Scala versions. I did use NSV with JavaScript, but parsers were ad hoc in projects' code. -->

