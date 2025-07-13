# Sales Pitches

Different arguments would appeal to different audiences, so in no particular order.

## For Vim and Vim-like tool users

The row separation with `\n\n` matches exactly Vi's notion of a "paragraph".
As such, you can move down `<n>` rows with `<n>}`, delete a row with `dap`, move to n-th cell with `<n>j`, etc.
CSV, JSON, and the rest would generally require additional setup to navigate.

## For tool developers

If your favourite tool does not have support for NSV, which at the time of writing it definitely doesn't, you can write it yourself at complexity of extending said tool.
The format is sufficiently simple to be processable with decent performance even by most naive implementations.

## For Obsidian and similar tool users

This one may be somewhat unexpected, but consider this.
If you define a matching rule and ignore paragraphs/rows that do not match it, you could use any and all tooling that works with NSVs to embed data directly in your notes at minimal overhead.
If you use the frontmatter, you'd have to handle it yourself first, but the rest of a file can be seen as valid NSV in most cases.

## For people who have to work with non-tech people

It is not uncommon to have some prepared data (e.g. lookup tables, exercise collections, localizations) that is never updated by an application user OR developer, but rather by people who come from the business or business domain side of things.
You would typically want to have a single source of truth for such.
What would that be?
A database? You'd have to then give the maintainers some form or UI to do the updates and deal with versioning on your own. And with visibility of the snapshot.
A CSV? With that you could indeed give people text to edit, they could even load that up in Excel and… but imagine the git diffs. Navigating one is not a pleasure either.
A YAML? A YAML may indeed have fewer problems with diffs, though the structure is hierarchical by nature, and overall more verbose for a table.

## For simple ADT encoding

NSV's variable-length rows allow encoding sum types with positional constructor arguments.
E.g. use the first element as discriminator, remaining elements as ordered constructor parameters.
Though anything that can encode/decode to a sequence of strings would do.
More compact than JSON's verbose field naming, cleaner than protocol buffers' `oneof` constructs.
