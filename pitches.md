# Sales Pitches

Different arguments would appeal to different audiences, so in no particular order.

## For Vim and Vim-like tool users

For a table, the row separation with `\n\n` matches quite neatly Vi's notion of a "paragraph".  
As such, you can move down `<n>` rows with `<n>}`, delete a row with `dap`, move to n-th cell with `<n>j`, etc.  
CSV, JSON, and the rest would generally require additional setup to navigate.

## For tool developers

If your favourite tool does not have support for NSV, which at the time of writing it definitely doesn't, you can write it yourself at complexity of extending said tool.  
The format is sufficiently simple to be processable with decent performance even by most naive implementations.

## For Obsidian and similar tool users

This one may be somewhat unexpected, but consider this.  
If you define a matching rule and ignore paragraphs/rows that do not match it, you could use any and all tooling that works with NSVs to embed data directly in your notes at minimal overhead.  
If you use the frontmatter, you'd have to handle it yourself first, but the rest of a file can be seen as valid NSV in most cases.
<!-- This happens to be what I was doing for years prior to realising I could formalise this as a data format. -->

## For people who have to work with non-tech people

It is not uncommon to have some prepared data (e.g. lookup tables, exercise collections, localizations) that is never updated by an application user OR developer, but rather by people who come from the business or business domain side of things.  
You would typically want to have a single source of truth for such.  
What would that be?  
A database? You'd have to then give the maintainers some form or UI to do the updates and deal with versioning on your own. And with visibility of the snapshot.  
A CSV? With that you could indeed give people text to edit, they could even load that up in Excel and… but imagine the git diffs. Navigating one is not a pleasure either.  
A YAML? A YAML may indeed have fewer problems with diffs, though the structure is hierarchical by nature, and overall more verbose for a table.

## For ADT lovers

NSV's variable-length rows allow encoding sum types with positional constructor arguments.  
E.g. use the first element as a discriminator, remaining elements as ordered constructor parameters.  
Though anything that can encode/decode to a sequence of strings would do.  
More compact than JSON's verbose field naming, cleaner than protocol buffers' `oneof` constructs.

## For performance-aware people

The format uses no quotation.  
Seeking the end of a cell is as simple as looking for the next newline char, same for the row ends.  
This alone makes the format far more amenable to parallel processing than most of its direct alternatives.

## For web devs

Think of debugging data flows.  
You've got the tooling for your JSONs but just consider: you won't need tooling for this one to be readable or to edit.  
You'd have to have per-data type encoders and decoders, sure, but it will not be putting more pressure on the wire, and would be far more readable for its use cases.

## For avid LLM users

In my experience, LLMs tend to grasp the nature of the encoding after just reading head of an encoded file.  
They sure miss details of the escaping (we did not make it into training data yet), but giving them just the three files in this repo, or any one of the implemenations, has been enough to get them up to speed.  
And these are short, well worth the tokens.

