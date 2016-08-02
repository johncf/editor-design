# In search of a better text editor, by design

Blueprints of an imaginary text editor!

## Table of Contents

- [Introduction](#introduction)
- [Design Goals](#design-goals)
- [Non-goals](#non-goals)
- [Terminology](#terminology)
- [Architecture overview](#architecture-overview)
  - [Concerns of Core](#concerns-of-core)
  - [Concerns of Shell](#concerns-of-shell)
- [Buffer management](#buffer-management)
  - [Handling huge files using `mmap`-ed buffers](#handling-huge-files-using-mmap-ed-buffers)
- [Typesetting](#typesetting)
- [The Modes](#the-modes)
- [The Cursor](#the-cursor)
  - [The Cursor Block](#the-cursor-block)
- [Text Objects and Cursor motion](#text-objects-and-cursor-motion)
- [Regular expressions](#regular-expressions)
- [Performance improvements through indexing](#performance-improvements-through-indexing)
  - [Real lines](#real-lines)
  - [Virtual lines](#virtual-lines)
  - [Bracket pairs](#bracket-pairs)
  - [Syntax highlighting](#syntax-highlighting)
- [Plugin architecture](#plugin-architecture)
  - [Capabilities](#capabilities)
- [Other design articles](#other-design-articles)

## Introduction

This was born out of the fascination I have towards text editors and see how I
would design one, borrowing many great ideas from our vast universe. The focus
was on making the whole thing as asynchronous and non-blocking as possible, and
consequently allowing stuff to be run in parallel.

This repository is [dedicated to the public domain (CC0)](LICENSE).

[design-articles]: #other-design-articles

## Design Goals

- Robust: can't afford lose stuff
- Performant: quick and non-blocking
- Modular: independent modules with clean interfaces
- Extensible: where the fun begins!

## Non-goals

- Define a new scripting language
- Support for proportional font rendering
- Accurate Unicode rendering (see [Typesetting](#typesetting))
- Multiple cursors

## Terminology

- Buffer: Represents the contents of the file being edited.
- Mode: An editor state (Normal, Insert etc.)
- Frame: An area where text is rendered, with dimensions in rows x columns

## Architecture overview

The editor has two main components: Core and Shell.

![Architecture diagram](https://critiqjo.github.io/editor-design/arch.svg)

### Concerns of Core

- Manage a single file
  - File IO
  - Buffer management (incl. undo history)
- Typesetting (constrained to frame dimensions)
  - Support for multiple views, with only one active view representing the
    current cursor position (and mode?)
  - Cursor movements and scrolling
- Text editing operations

### Concerns of Shell

- Visual rendering
- User input handling
  - Key mappings (The Core has no concept of keys)
  - Macros
- Syntax highlighting
  - Parser implemented as a separate independent component
- Text completion
- Search and replace
- Integration with external tools

## Buffer management

The data structure used to represent and manage text is the very core of an
editor. It should be so designed as to provide these two features:

- Efficient and threadsafe snapshots: for processing text concurrently
- Efficient diffing of two versions: for efficient reuse of old processed data

A copy-on-write [Rope][rope-wiki]-like data structure should be used to
represent the contents of a file. All text (including all history of changes)
should be stored in a single thread-safe, append-only list (TODO describe), and
the leaves of the rope should just store offsets in the list and length of the
chunk, except when size of the chunk is less than 16 bytes which may be
directly embedded. Reasons for the choice:

- Create thread-safe view of the contents with O(1) cost
- Having almost all chunks being represented using an offset and length, diff
  of two versions in history can be found in roughly O(n^2) space and time with
  a slightly modified LCS algorithm, where n is the number of chunks in the
  rope (not file size). For asynchronous syntax highlighting, efficient diffing
  can prove very useful.

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

## Typesetting

- **All Unicode code points are separately and visibly rendered.**
  - Including non-printing characters
- Half-width characters take up 1 column each.
- Full-width characters take up 2 columns each.
- [Combining characters][combining-chars] that take up extra space are rendered
  normally, and the rest are combined with whitespaces (with distinct
  highlighting), so that they are all separately addressable.
  - [Normalization][unicode-norm] should be suggested where applicable.

[combining-chars]: https://en.wikipedia.org/wiki/Combining_character
[unicode-norm]: https://en.wikipedia.org/wiki/Unicode_equivalence#Normalization

## The Modes

The Core has 2 main states/modes:

- Insert: When character inputs are inserted directly
- Select: When a range of text is selected (character inputs replaces the
  selection transitioning to Insert mode)

Additionally, there are 2 types of operator sub-states, when it waits for an
appropriate operand:

- TextObj: Waiting for a text object
- CharStream: Waiting for one or more characters

The Shell will simulate two additional modes:

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

In Select/Visual mode, the Cursor Block has no functional significance, and is
always placed inside the range at the active end (note: `length != 0`). Also
note that when `length` changes from 1 to -1 or vice versa, the Block moves
only by one column.

## Text Objects and Cursor motion

A text object is a range of text whose contents/endpoints are constrained by
certain rules. The text objects discussed in this section are located near or
around the cursor position. Therefore, in the table below, "Start" refers to a
point at or before the cursor, and "End" refers to a point at or after the
cursor. `.` represents the cursor position itself, and `/...` represents a
regular expression.

When "Start" or "End" is `.`, it can be used to define a cursor motion. If
"Start" is `.`, the cursor moves forward, and otherwise, backwards. "Block pos"
is the position of Cursor Block relative to the Cursor, after the movement has
been made.

| Text object   | Keymap    | Start       | End         | Block pos  |
|---------------|-----------|-------------|-------------|------------|
| word          | `w`       | `.`         | `/\b\w/s`   | right      |
| word-rev      | `b`       | `/\b\w/s`   | `.`         | right      |
| word-end      | `e`       | `.`         | `/\w\b/e`   | left       |
| word-end-rev  | `ge`      | `/\w\b/e`   | `.`         | left       |
| word-inner    | `<op>iw`  | `/\b\w/s`   | `/\w\b/e`   | (NA)       |
| find-char     | `f{char}` | `.`         | `/{char}/e` | left       |
| find-char-rev | `F{char}` | `/{char}/s` | `.`         | right      |
| till-char     | `t{char}` | `.`         | `/{char}/s` | left       |
| till-char-rev | `T{char}` | `/{char}/e` | `.`         | right      |
| left          | `h`       | `/./s`      | `.`         | unchanged? |
| right         | `l`       | `.`         | `/./e`      | unchanged? |
| up            | `k`       | (custom)    | `.`         | unchanged? |
| down          | `j`       | `.`         | (custom)    | unchanged? |
| up-virt       | `gk`      | (custom)    | `.`         | unchanged? |
| down-virt     | `gj`      | `.`         | (custom)    | unchanged? |
| ...           |           |             |             |            |

Note: up and down text objects are captured with the help of [indices](#real-lines).

Below is an illustration of cursor movements and `d`-commands involving them
that are different from (but more intuvitive than) how Vim behaves.

Also note that the position of cursor _block_ does not play any role in
determining the outcome of `d`-commands.

![Cursor movements](https://critiqjo.github.io/editor-design/cursor.svg)

## Regular expressions

The Shell will provide two simple mechanisms involving regular expressions:

- Search
- Search and replace
- Search and replay (a macro)

Apart from these, the user must be encouraged to use external tools such as
`awk` or `sed`. To that end, the user may be provided with a feature to
"preview" such commands. A "preview" is just a non-blocking way to partially
view the processed result, also trying to minimize the execution by reading in
only just enough to fill the current frame (then kill/suspend the subprocess).

## Performance improvements through indexing

An entry in an index has four major fields:

- Offset: Byte offset of this entry from the beginning of file
- Version: Version of the buffer using which this entry was created
- State: The state of the parser or something when this entry was created
- Data: Something useful

When some text is added or deleted from the buffer, the data structure of the
index should be such that offsets can be efficiently updated. The entries
following the point of change need not be deleted. In fact, a few consecutive
changes in text might make most of the indexed data useful again (for example,
opening a curly brace, entering some text and then closing it later, without
affecting other pairings).

This is why we need the State entry to make the indexing lazy. Before using the
Data, check if the current state at that offset is indeed the same as when it
was indexed.

The indices will be coarse grained or non-existent as we move farther from the
Cursor. We will be making a few approximations in case we need data from those
parts of the buffer.

### Real lines

A real line is a range of text that appears between two line-breaks or between
a line-break and the beginning/end of file.

- State: Nil
- Data: Total width?

### Virtual lines

A virtual line is a range of text that is type-set in a single row.

- State: Relative byte offset to a character in the same (real) line which is
         rendered at column 0
- Data: Character offset (w.r.t the first character in the line)

### Bracket pairs

- State: Nil
- Data: Offset of the matching pair

To check whether an entry is valid/useful in the current version, check whether
any changes were made between the bracket pair and if so, check whether the
changes tinkered with brackets (of same type).

Indexing bracket pairs may not be worth the effort unless we are handling large
json file, or something similar.

### Syntax highlighting

- State: Something regarding the state of the parser
- Data: Highlight group

This part is very open ended and really depends on the type of parser used.
Furthermore, this component can be swapped with a language-specific semantic
checker to provide very rich highlighting (e.g. highlight errors, using symbol
table to determine the type of an identifier, etc.).

## Plugin architecture

There are two types of plugins:

- **Core plugin**: Synchronous hooks - compiled along with core
- **Shell plugin**: Asynchronous - dynamically loaded through scripts

Maintain a centralized index of plugins, and also an option to download a
pre-compiled binary of custom combination of core plugins (through on-demand
compilation and caching?).

### Capabilities

Also maintain a centralized index of (versioned) _capabilities_. A capability
defines an interface through which a certain task can be accomplished. A plugin
may provide zero or more capabilities. Thus capabilities are a layer through
which plugins may interact with and exploit each other. Capability is a concept
existing only in the Shell.

## Other design articles

- [GNU Emacs Internals](https://www.gnu.org/software/emacs/manual/html_node/elisp/GNU-Emacs-Internals.html#GNU-Emacs-Internals)
- [Architecture Analysis and Repair of Open Source Software (page 7 onwards)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.46.6702&rep=rep1&type=pdf)
- [Vim, an open-source editor (section 'Storing text' onwards)](http://www.free-soft.org/FSM/english/issue01/vim.html)
- [Kate internals: Text buffer](https://kate-editor.org/2010/03/03/kate-internals-text-buffer/)
- [Vis: Text management using a piece table](https://github.com/martanne/vis/blob/master/README.md#text-management-using-a-piece-tablechain)
