# Newline-separated values

## Why? (advantages)

- Better Git diffs
- More flexibility for annotations
- Simpler implementations
- Better navigation in vim-like tools

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
But I told it to Claude and he said he knows no encoding format similar to what I was describing, and that brings us here.
The more we iterated on it, the more it made sense, and the less made its absence.
So here I am, bringing yet another encoding format into this accursed development world of today.

## Is this to CSV what CSV is to TSV?

Here, naming may be somewhat confusing, but no.
Iterating on it a bit made me realise that I'd have to reimplement all the common tooling, and at that point, why would I bring with me all the design baggage of those formats?

So first, NSV is not a "table" format.
What is encoded is a "sequence of sequences".
Exactly one layer of nesting, and if all the nested ones happen to have same length — that's a table!
(I'm fending off the desire to call the nested ones "subsequences" but the word exists so I shall suffer.)

Second, I am not going to allow arbitrary text.
Making *SV formats deal with raw text was a horrible decision, and people duly abused it as one would expect.
This one shall not make that mistake.
You wish to have arbitrary text?
Sanitize your newlines as you will.

Lastly, I indend to allow for somewhat richer metadata.

## Specification

Never wrote a rigorous one, so bear with me or better yet suggest improvements.

Starting from the top, there is a split into header with metadata and the data itself.

The metadata is not optional and ends at first `\n---\n` encountered in file.
One may recognise this as being heavily reminiscent of Markdown frontmatter, except for lacking opening horizontal rule.
Since we're starting fresh there's no requirement to be able to parse data that may not contain it, and I would want at least the version to be there for future compatibility.

The general parsing rule for the data paragraphs is as following
1. Split on double newline to get rows
2. Split each on single newline to get values
3. Apply type conversions, if you need any
4. Pat yourself on the back

The format itself is data type-agnostic.
Everything is a string, and you are to make sense of them yourself, however you see fit.
For practical applications, parsers would normally tightly integrate with converters, but deciding on what strings mean what is not up to this spec with a sole exception: the empty field token, `\`.

Fields that contain no value, and would normally be represented as an empty string, must be explicitly represented with a single backslash.
While the specification does not (yet) specify how escaping should be done, implementers are encouraged to use `\n` and co., `\` would then correspond to an invalid string that would never be encoded by backslash-escaped encoding.
As to why the token is needed in the first place: we need it to make parsing unambiguous while retaining the simplicity of implementation.

Empty paragraphs, i.e. sequences of 2N consequtive newlines, are to be considered valid sequences of zero elements.
A sequence of 2N+1 newlines is to be considered invalid.
Arbitrary sequences of consecutive newlines are allowed in the header.
At most one newline is allowed immediately after the header-data separator.

I see now that I can't really write a spec and end up yapping instead.
Well, then, here's an example.

```nsv
v1
table:4
# Yappy yappy yap
---
first
group
of
elements

second
group
of
elements

one
with
\
field
```

Each line in the header is taken as a literal string.
The order does not matter, with the sole exception of version label, which shall be first (so as to aid the parser with interpreting the rest of metadata lines).

Lines that start with `//`, `#`, or `--` **do not and never will** have any special meaning under this standard.
As such, you can use those to comment or keep any amount of metadata.

Some strings would then be further interpreted to have special meaning.
The only special string formats used right now are a literal label and `key:value`, both being alphanumeric, but one should not assume it will be limited to just that.

As for those with special meaning:
- `v:<major>.<minor>` indicates the version number (only `v:1.0` is valid at the time of writing) and is **the only required parameter**
- `table:<number>` indicates that the file is supposed to represent a table of `<number>` columns; if a paragraph has a mismatched number of fields, it is to be considered invalid

### Possible future extensions

Likely
- naming columns in metadata
- specifying column types in metadata

Under consideration
- heredoc-like syntax for multiline inclusions
- allowing a single multiline field per element if the schema is explicitly provided
- comment support in the data paragraphs themselves

### Version compatibility considerations

The specification itself only uses MAJOR.MINOR format.
The implementations are encouraged to match major and minor version of the highest specification version they support, leaving patch and other version sections up to them.

For an implementation to "support" a given spec version, it must
1. Support **all** required labels
2. **If** it interprets any optional labels, it must match the semantics described in spec exactly

Major version bumps are reserved for changes that would break existing implementation.
A breakage means "the tool cannot interpret the file while omitting it's optional labels".
This would normally be reserved to changes in required label list or general parsing logic changes.

Minor version bumps would then correspond to changes in optional features.

