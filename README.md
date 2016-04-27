# In search of a better text editor, by design

Blueprints of a better engineered text editor!

## Introduction

There have been too many text editor implementations. Some have their designs
published openly<sup>[\[1\]](#other-design-articles)</sup>. There are great
ideas hiding out there, waiting to be picked up! This repository is an attempt
to bring together innovative ideas to design the greatest text editor in
existence!!!

I wish to maintain a nearly complete blueprint of a text editor.

This article is released under the terms of CC0 license.

## Design Goals (vague)

- Robust
- Modular
- Performant
- Cross-platform viable
- Extensible

## Non-goals

- Define a new scripting language
- Support for proportional font rendering
- Multiple cursors

## Terminology

- Buffer: Represents the contents of a file being edited.
- Mode: An editor state (Normal, Insert etc.)

## Separation of concerns

The entire editor is broadly separated into two components: Core and UI.

### Concerns of Core

- Manage a single file
  - File IO
  - Buffer management (incl. undo history)
- Typesetting (constrained to given dimensions -- rows x cols)
- Command handling
  - Cursor movement and scrolling
  - Text editing operations
  - and more...

### Concerns of UI

- Visual rendering
- User input handling (keyboard and mouse)
  - Key mappings (The Core does _not_ maintain any keymaps)
- Syntax highlighting
  - Parser implemented as a separate independent component
- Text completion

## Typesetting

- Half-width characters take up 1 column each.
- Full-width characters take up 2 columns each.
- Combining characters does not take up any space on its own _if_ there is a
  "combinable" character preceding it.
- All other characters (control characters, zero-width space, etc.) should be
  "specially" rendered.

## The Modes

The Core identifies three modes:

- Insert: When printable keys are taken literally
- Select: During text-selection (printable keys replaces the selection)
- Operator: When an operator waits for operand(s)

The UI will simulate two additional modes:

- Normal: For easy access to operations in Insert mode
- Visual: For easy access to operations in Select mode

## The Cursor

The Cursor is defined as a byte range `(offset, length)`, with the restriction
that the endpoints must be at a character boundary.

`(0, 0)` means that the cursor is positioned before the first byte.

In Insert mode, `length` is always equal to `0`.

In Select mode, the active end, where the cursor movements are applied, is at
`offset + length`. Thus it is valid for `length` to be negative, but not zero.

### The Cursor Block

In Insert/Normal mode, the Cursor Block marks the character that is focused by
the Cursor, on which character-wise operations are applied. Depending on
context, it may be to right of the Cursor, or to the left. We will discuss more
about this in Text Objects section.

In Select/Visual mode, the Cursor Block has no value, and can be placed at the
active end inside the range. Note that when `length` changes from 1 to -1 or
vice versa, the Block only moves by one column. (TODO illustrate)

Cursor motion is based only on the Cursor position (not the Block).

## Text Objects and Cursor motion

All cursor motion is defined in terms of text objects.

## Buffer management

A copy-on-write [Rope][rope-wiki]-like data structure should be used to
represent the contents of a file. All text (including all history of changes)
should be stored in a single vector, and the leaves of the rope should just
store offsets in the vector and length of the chunk. Reasons for the choice:

- Create thread-safe view of the contents with O(1) cost
- All chunks can be represented using an offset and length. Thus diff of two
  versions in history can be found in O(n^2) space and time with a slightly
  modified LCS algorithm, where n is the number of chunks in the rope (not file
  size). For asynchronous syntax highlighting, efficient diffing is necessary).

[rope-wiki]: https://en.wikipedia.org/wiki/Rope_(data_structure)

### Handling huge files using `mmap`-ed buffers

Having an in-memory vector of all text, when handling huge files, could be
cumbersome. Therefore an opt-in feature to memory map a file would be nice. In
this scheme, a chunk in the rope can be represented by either an offset to an
in-memory vector, or an offset to a memory mapped vector. But memory mapping
has certain limitations and platform-specific quirkiness. Thus a few features
may have to be disabled, and/or a few restrictions have to be brought in when
handling mmapped buffers.

If undo-history has to be preserved, `mmap`-ed files should not be overwritten
while saving. Instead, create a new file with name `<file>.N` (`N` > 0).
(Alternatively, in Linux systems, the old file can be safely moved to
`<file>.N` while `mmap`-ed and the new file can be named `<file>`. Not sure if
Windows supports this.)

Warn that large files (and its versions) should not be modified in parallel
from another process; which may cause UB!!

## Regular expressions

Interactive and rich similar to Sam's.

## Performance improvements through indexing and approximations

An entry in an index has three major fields:

- Offset: Buffer byte offset of this entry
- State: The state of the parser or something when this entry was created
- Data: Something useful

When some text is added or deleted from the buffer, the data structure of the
index should be such that offsets can be efficiently updated, but the entries
following the point of change need not be deleted. In fact, a few consecutive
changes in text might make most of the indexed data useful again (for example,
opening a curly brace, entering some text and then closing it later).

This is why we need the State entry to make the indexing lazy. Before using the
Data, check if the current state at that offset is indeed the same as when it
was indexed.

The indices will be coarse grained or non-existant as we move farther from the
Cursor. We will be making a few approximations in case we need data from those
parts of the buffer.

### Line-breaks

- State: Nil
- Data: Nil

### Virtual lines

- State: Relative byte offset to a character in the same (real) line which is
         rendered at column 0
- Data: Nil

### Bracket pairs

- State: Depth (number of unclosed brackets)
- Data: Offset of the matching pair

### Syntax highlighting

- State: State identifier of the parser state machine
- Data: Color

## Plugin architecture

Two types:

- **Core plugins**: Synchronous hooks - compiled along with core
- **UI plugins**: Asynchronous - dynamically loaded through scripts

Maintain a central list of plugins and option to download custom combination of
core plugins using on-demand compilation (with caching) 

### Capabilities

There should also be a central list of (versioned) _capabilities_. A capability
defines an interface through which a certain task can be accomplished. A plugin
may provide zero or more capabilities. Thus capabilities are a layer through
which plugins may interact with and exploit each other. Capability is a concept
existing only in the UI layer. Is there a possible use-case for Core plugins
interacting with each other?

## Other design articles

- [GNU Emacs Internals](https://www.gnu.org/software/emacs/manual/html_node/elisp/GNU-Emacs-Internals.html#GNU-Emacs-Internals)
- [Architecture Analysis and Repair of Open Source Software (page 7)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.46.6702&rep=rep1&type=pdf)
- [Vim, an open-source editor (from section 'Storing text')](http://www.free-soft.org/FSM/english/issue01/vim.html)
- [Kate internals: Text buffer](https://kate-editor.org/2010/03/03/kate-internals-text-buffer/)
- [Vis: Text management using a piece table](https://github.com/martanne/vis/blob/master/README.md#text-management-using-a-piece-tablechain)
