## Bad Ideas

Aka things I have considered and eventually considered against.
I'm writing this to keep track of why certain decisions were made, for I am starting to forget.
A secondary goal is dealing with the feeling that this whole thing is too simple to even bother specifying anything.

### General thoughts

A common theme is keeping the things simple for potential implementers.
The "simple rule" should allow for trivial parsing of the body with zero need for lookahead/lookbehind anywhere.

### Not having metadata, or having optional metadata

I think I've seen enough formats that suffer because of compatibility being mishandled.
I have zero confidence that I'd be able to nail it myself, but at least demanding a version should smoothen things up.

As for optionality of the header, if it was optional then I'd end up needing to resort to an ugly solution like that of Markdown frontmatter to know whether I'm parsing the header or the body already.

In principle, allowing metadata being passed separately, letting a parser deal with "headless" NSV should be trivial down the line, so it seemed like a safe bet.

### Using CSV-like escaping

Gladly, wasn't much of a detour: CSV's approach to escaping multiline and commas was likely driven by existing baggage which a new format would not share.
With newline being the only special symbol to consider in the body, one would only have to somehow deal with multiline strings being encoded, no escaping commas then escaping the quotes used to escape commas then… bruh.

### Not using an empty field token

Since I started off trying to replace CSVs, the original intention was to only handle table data, i.e. sequences where you'd know the number of elements in each to be equal.
At that point, having simpler parsing and better Git diffs were the only goals.

It quickly dawned on me that the readability of such format would be abysmal, just imaging eyeing row separation vs multiple empty fields, which are oh so common in real-life tables.
It would also make absolutely necessary specification of number of fields in the header, as there would be no way to tell that from the data itself.

The `\` as empty cell token was the first and only one I have considered.
It immediately made sense since it's "unoccupied" in the most common escaping style, easy to type, and is visually lightweight.
And looks like line continuation in many languages. And has a tad of "nothing" vibe. And how often do you have that as a separate cell in table, anyway?

### Per-cell inline comments

Now, this is a funny one, and I did not give up on it, it just clearly has no place in a `1.0`.
In principle, if any cell occupies its own line, one could end said line with a comment, allowing the data file itself have annotations on cell level.
Why on Earth would you want that?
Remember that the originally pain point was dealing with lookup tables. In a relational database setup, it's not uncommon for them to be normalised.
Now, while the whole point (well, a good portion) of normalisation is not having the same thing in multiple places, it would aid the readability greatly, if instead of just an ID of a, say, product category that is described in a neighbouring table was available right there in the data file itself.
You could even automate annotations like that!
It would of course be very troubling while the format is completely data type-agnostic, but once you have tables and can specify field types in the header and can specify a comment token in the header, you would be able to have painless commenting on at least numeric fields, ISO timestamps, UUIDs, alphanumeric identifiers, and anything else that is truly arbitrary.

<!-- Funny how the paragraphs are getting longer and longer. -->

### Using heredoc-like style for multiline strings

I completely gave up on this one, but it did not look like a bad idea from the outset.
Having the special token defined in the header, one could continue using a fairly naive parsing approach (having just one special state for when inside a heredoc). Readability of those would arguably be far better than that of lines with length in hundreds or thousands.

The multiple reasons that eventually made me reconsider are
- I just really care much more about tables/sequences, and arbitrary text is a second-class citizen, and that when I'm generous
- It would make paragraph-based navigation impossible without additional setup

It is at about this point that I gave up on the idea of allowing per-cell comments to be on a separate lines, or having ignored lines.

### Focusing on tables

And about here I have realised that now with the empty field token I could relax the requirement of having equal row lengths.

This, in turn, introduced a new not-so-funny edge case that drove some of the decisions down the line: empty rows.
It would be horrible to not be able to encode an empty sequence or require a special token to do so, and using just `\` would not work since that's a single empty element sequence (think `[]` vs `[""]`).
At about this point, after hitting the wall a few times, I decided to stick to what I refer to as "simple rule" throughout these docs.

Now, an interesting thing about variable-length rows is that one could in principle encode a sequence of elements from an ADT this way.
That is, instead of having a sequence of objects of an exact same type, you could hook a decoder that would match each row to an appropriate class in a closed hierarchy, with no weirdness like passing empty fields that are not required by some subtypes, etc.

### Being smart about metadata

Having key-values there, or just labels, or processing it in any way beyond extracting what's relevant for further parsing was another detour.
While richer, frontmatter-like YAMLy metadata could allow some neat things, it would also complicate life a lot for what is intended to be a simple-to-handle format.
The ability to double-purpose a Markdown file as an NSV (or even multiple NSVs at the same time) came up strongly in here.

What matters is that by the time we reach the end of the header we know all we need to know to parse the rest of the file.
If the order of meaningful keys/labels is not important, we can safely preserve all the original header lines in the exact order they were provided.

At the same time, providing guarantees that nothing prefixed with common symbols used for comments or typical data encodings would mean that the end-users of the data files could in principle keep any amount of description together with the data itself.
This could, though not part of the original intention, be useful if the lookups are fed to an LLM (though they would probably fare better with a format which specifies field name for each cell).

At first, I've though of having metadata really laconic, like `v1,table` or `v1,table:2,columns:[col1,col2]`.
But why commas? And what's the deal with the brackets?
It would only make sense that the newline-based format uses newlines to separate items in the header, too.
But the appeal of using CSV-style list of names for columns…
And yet, I decided against it. Problem being same of that with CSV, what if column names are arbitrary text?
It's not uncommon that they would contain commas or even newlines, and then we again fall into the whole escaping hell.

### Type-awareness

For a text-based format where everything is already a string anyway, it makes little sense to try and dictate how numbers and the rest should be encoded.
While leaving that entirely up to the implementations would likely lead to them using different encodings for common data types, trying to address it here would enormously bloat the scope.
The intent here is to handle a 2D structure conversion to a 1D text format and back.
It would of course be useful to provide an ability to specify types so as to allow parsers to decode straight to the desired types, hook custom decoders, etc.
But the encodings themselves, even by reference to existing standards? Out of scope.

### Versioning and compatibility

This one gave me, by far, most trouble.
The general advice by people and LLMs alike is "use semver". Semver what?
After struggling for a bit, the way I see it that there's (at least) three parts here
- the specification itself
- individual implementations of it
- data files produced by said implementations

So who exactly is promising backward and forward compatibility to whom? I'm confused.

### Not including explicit escaping rules

Did not strike me as obvious until I wrote tests, but not specifying how newlines inside cells should be escaped would lead to a lot of variability between implementations and be a pain to deal with down the line.

Now, given that the `\`-based escaping seems preferable, the minimal set of escape sequences should be one for the newline and one to escape the escaping itself, i.e. `\n` and `\\`.

A bigger question is how to handle what appears to be an unrecognized sequence.
The options are at least
1. fail on unrecognized sequence
2. ignore, discard the escape character
3. ignore, let the escape character through

The first one appears too painful, even if failure were limited to a cell, what to do with it, warn?
The second one… is fine, that's what renderers use often, but what if the sequence was there for a reason? How would the reader know that there was an escape character, to maybe fix the issue on their side?
So I'll settle for the last.
In principle, that would mean that the parsers would be able to encode and decode more escape sequences if they choose so, which is good.
The spec would have to clearly indicate that the parser should not fail because of escape sequences it is unfamiliar with.
