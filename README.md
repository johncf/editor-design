# In search of a better text editor, by design

Blueprints of an imaginary text editor!

## Table of Contents

- [Introduction](#introduction)
- [Design Goals](#design-goals)
- [Non-goals](#non-goals)
- [Terminology](#terminology)
- [Architecture overview](#architecture-overview)
  - [Concerns of the Shell](#concerns-of-the-shell)
  - [Concerns of a Buffer](#concerns-of-a-buffer)
  - [Concerns of the Filer](#concerns-of-the-filer)
  - [Concerns of the Skin](#concerns-of-the-skin)
- [Text management](#text-management)
  - [Handling huge files using `mmap`](#handling-huge-files-using-mmap)
- [The Cursor](#the-cursor)
  - [Vim-like implementation](#vim-like-implementation)
  - [Text Objects and Cursor motion](#text-objects-and-cursor-motion)
- [Regular Expressions](#regular-expressions)
- [Scroll Bars](#scroll-bars)
  - [With word-wrap (vertical scrolling)](#with-word-wrap-vertical-scrolling)
  - [Without wrapping (horizontal scrolling)](#without-wrapping-horizontal-scrolling)
- [Syntax highlighting](#syntax-highlighting)
- [Plugins](#plugins)
- [Capabilities](#capabilities)
- [API](#api)
- [Other design articles](#other-design-articles)

## Introduction

This was born out of the fascination I have towards text editors and see how I
would design one, borrowing many great ideas from our vast universe. The focus
was on making the whole thing as asynchronous and non-blocking as possible, and
consequently support concurrent jobs (plugins).

This repository is [dedicated to the public domain (CC0)](LICENSE).

[design-articles]: #other-design-articles

## Design Goals

- Reliability - Zero risk of losing data
- Responsive - Zero blocking operations (at worst, jobs must be interruptible)
- Modular - Largely decoupled modules with clear, well-documented interfaces
- Extensible - Easily customizable with plugins

## Non-goals

- 100% accurate rendering of complex Unicode scripts
- In-process extensions through a scripting language

## Terminology

- **Buffer** is a data structure that handles the most basic text and history
  manipulations for a single file.
- **The Shell** is the glue between the user interface, plugins and buffers.
- **The Filer** is a component of the Shell which handles all disk I/O.
- **The Skin** is the user interface.

## Architecture overview

![Architecture diagram](img/arch.svg)

### Concerns of the Shell

- Connect user interface with buffers
- Cursor book-keeping: cursor position, text selection
- Scrolling book-keeping: total lines, line wraps, visible text
- Key mappings (incl. macros)
- Extensions management
  - Communication with external tools
  - [Capability](#capabilities) assessment
  - Syntax highlighting (via extensions)
  - Text completion (via extensions)
- Search and replace

### Concerns of a Buffer

- [Text management](#text-management)
- Handle text editing operations: insert, delete
- Handle history operations: undo, redo
- Serve text in chunks on-demand.

### Concerns of the Filer

- Handle all disk I/O.
- Run in a separate thread to avoid blocking.

### Concerns of the Skin

- Text rendering (incl. typesetting)
- User input handling

## Text management

The data structure used to represent and manage text is the very core of an
editor. The following design was heavily influenced by [the CRDT model][].

The buffer contents, along with the history of changes, will be captured by the
following entities:

- _A versioned Text Wall,_ which is an ever-growing (never-shrinking) sequence
  of characters, where any text insertions are inserted into Text Wall
  "in-context" and text deletions ignored. This is probably best represented by
  a [persistent rope-like data structure][rope-wiki].
- _A linear history of Text Wall,_ which is a sequence of revision ids and the
  corresponding interval(s) of inserted text. This will be used for computing
  the index position w.r.t the latest revision of Text Wall, given an index
  w.r.t a past version of the wall.
- _Text Wall Mask,_ which represents the actual buffer contents. A mask is
  tied to a specific revision of Text Wall, and contains a sequence of intervals
  which "masks" Text Wall to obtain the buffer contents at that point.
- _A linear history of actions on the buffer,_ where an action is a set of
  `show`-s and `hide`-s.  A `show` or a `hide` is a change brought to the
  previous version of the mask. When a sequence of undos is followed by an edit
  operation, squash those undo actions into one history action followed by a
  negative of that action, and apply the edit operation on top of it.

[the CRDT model]: https://github.com/google/xi-editor/blob/6683f20/doc/crdt.md
[rope-wiki]: https://en.wikipedia.org/wiki/Rope_(data_structure)

Text Wall and Text Wall Mask need to sophisticated enough to be shared
concurrently (in a copy-on-write manner). The other two data structures might
not require these features, since only a single version of them would need to
exist at any point.

This will enable implementing persistent undos feature very straight-forward,
but truncating undos to a few hundred actions so as to shrink an over-sized Text
Wall might be expensive, unless some additional indexing is performed.

### Handling huge files using `mmap`

Having an in-memory representation of all text, when handling huge files, could
be cumbersome. Therefore the user should be presented with the choice of using
memory mapping the file with certain limitations. Memory mapped files will be
opened only in read-only mode. When saving, the original file will not be
touched by default; instead, a patch-like file will be created which can be used
to construct the modified file from the original file at a later point. This
way, intermittent file-save operations can be very quick.

## The Cursor

The cursor is a concept defined by the Shell. It is minimally represented by a
tuple: `(l, o)` where `l` is the line number and `o` is the byte-offset on the
line. When a selection is being made, an additional tuple is needed to
represent the other end of the selection. To generalize, a cursor can be
thought of as a span from `(l1, o1)` to `(l2, o2)` where `*1 = *2` when no text
selection is made.

Multiple cursors may be implemented by allowing a set of disjoint spans to
co-exist. Keep in mind that allowing user to easily create a thousands of
cursors might result in a significant slow-down. The weaker solution of
rectangular selection (visual-block) in Vim would likely perform much better in
cases applicable to both.

In addition to the raw value, the shell exposes two UI components for the
cursor: the cursor beam and the cursor block. How or whether they are shown can
all be customized by plugins. The default implementation is described below.

### Vim-like implementation

The cursor has four basic states: Insert, Select, Normal and Visual.

In Insert/Normal mode, the Cursor Block marks the character that is focused by
the Cursor, on which character-wise operations are applied. Depending on
context, it may be to right of the Cursor, or to the left. We will discuss more
about this in the next sub-section.

In Select/Visual mode, the Cursor Block has no functional significance, and is
always placed inside the range at the active end of the selection span.

### Text Objects and Cursor motion

A text object is a range of text whose contents/endpoints are constrained by
certain rules. The text objects discussed in this section are located near or
around the cursor position. Therefore, in the table below, "Start" refers to a
point at or before the cursor, and "End" refers to a point at or after the
cursor. `.` represents the cursor position itself, and `/...` represents a
regular expression.

When "Start" or "End" is `.`, it can be used to define a cursor motion. If
"Start" is `.`, the cursor moves forward, and otherwise, backwards. "Block pos"
is the position of Cursor Block relative to the Cursor, after the movement has
been made. Cursor block prefers to be on the right of the cursor. There are two
built-in commands that depend on the position of the cursor block: `r` and `a`
(Note: `s` is redundant and not defined; `a` is the same as `i` if cursor block
is on the left and `li` otherwise). _All_ other commands work based on the
cursor position alone.

| Text object   | Keymap    | Start       | End         | Block pos  |
|---------------|-----------|-------------|-------------|------------|
| word          | `w`       | `.`         | `/\b\w/s`   | right      |
| word-rev      | `b`       | `/\b\w/s`   | `.`         | right      |
| word-end      | `e`       | `.`         | `/\w\b/e`   | left       |
| word-end-rev  | `ge`      | `/\w\b/e`   | `.`         | left       |
| find-char     | `f{char}` | `.`         | `/{char}/e` | left       |
| find-char-rev | `F{char}` | `/{char}/s` | `.`         | right      |
| till-char     | `t{char}` | `.`         | `/{char}/s` | left       |
| till-char-rev | `T{char}` | `/{char}/e` | `.`         | right      |
| char-left     | `h`       | `/./s`      | `.`         | right      |
| char-right    | `l`       | `.`         | `/./e`      | right      |
| line-up       | `k`       | (custom)    | (custom)    | right      |
| line-down     | `j`       | (custom)    | (custom)    | right      |
| line-prefix   | `^`       | `/^/`       | `.`         | right      |
| line-suffix   | `$`       | `.`         | `/$/`       | left       |
| row-up        | `gk`      | (custom)    | `.`         | right      |
| row-down      | `gj`      | `.`         | (custom)    | right      |
| word-inner    | `<op>iw`  | `/\b\w/s`   | `/\w\b/e`   |            |
| ...           |           |             |             |            |

Below is an illustration of cursor movements and `d`-commands involving them
that are different from how Vim behaves (but perhaps more intuvitive).

![Cursor movements](img/cursor.svg)

Note: `d`-commands are not affected by the position of cursor block.

## Regular Expressions

The Shell will provide three simple mechanisms involving regular expressions:

- Search
- Search and replace
- Search and replay (a macro)

Apart from these, the user must be encouraged to use external tools such as
`awk` or `sed`. To that end, the user may be provided with a feature to
"preview" such commands. A "preview" is just a partial view the processed
result in order to minimize execution time while composing the command.

## Scroll Bars

For smooth scrolling, under the assumption that all rows of text have the same
pixel height, we have two cases

### With word-wrap (vertical scrolling)

For each line, the Shell need to store the list of offsets where the line wraps
(for lines with only very few points of wraps, the Shell may simply store the
line numbers in a wrap-count-histogram list). This should be informed by the
Skin to the Shell after laying out a line. When the width of the window or the
font size changes, the Skin should inform the Shell of it so as to invalidate
the wrap-points cache.

There is no need to layout the entire text to estimate an approximate height of
the whole text. Once a screen-full of text is rendered, calculate the average
number of bytes per row (`n_ravg`), and the maximum number of bytes per row
(`n_x`), then the expected number of additional line wraps is:

    (n_bytes - n_lines*n_ravg)/n_m

The scroll range is `(0, n_rtotal)` where `n_rtotal` is the total number of rows
in the file. The current scroll value would be based on the current estimates of
how many rows there are prior to the first line currently displayed.

Now, regarding mapping between scroll value to text needed to be rendered.

- The Skin asks the Shell for text at `k`th row.
- The Shell replies with:
  - A range of text that is estimated to be the start of that row (if that line
    was previously rendered), or
  - A range of text from the start of the line that is estimated to come right
    before that row (along with the row number of the start of that line).
- The Skin then layouts the text starting from what was received from the Shell
  till it reaches the expected row, then start rendering whatever required.
- The skin tells the Shell about the points of wraps in layouted text.

### Without wrapping (horizontal scrolling)

When text wrapping is disabled, vertical scrolling is straightforward.
Horizontal scrolling on the other hand is slightly tricky in the presence of
very long lines. First off, we make the limits of the horizontal scrollbar to
only reflect the range of lines that is currently displayed. This is in best
interest for the user too since if there is one very long line somewhere on the
file, it shouldn't make it hard to scroll horizontally when working on a part
that is far away from that long line.

Previously, in case 1, the shell didn't need to deal with pixel-measurements,
but here we will. Another thing to note is that we already know the number of
bytes each line contains (or at least it can be calculated very quickly no
matter how long the line is).

Now, the way this works is:

- The Skin renders minimal amount of text that's needed to fill the screen (long
  lines need not be fully read).
- The Shell receives data from the Skin regarding how many pixels were required
  for rendering whatever ranges of bytes that were rendered.
- The Shell estimates the maximum width of lines that is currently in view,
  based on the above data, and tells the Skin about the limits of scrollbar.
- When user scrolls horizontally, the Skin asks the Shell for text at certain
  lines that are `k` pixels away from the start; and the Shell responds with a
  range of text that is known to start before `k` pixels from start (along with
  the actual starting pixel position of that range.)

For each line, the Shell needs to store a list of offsets in bytes and the
corresponding offsets in pixels (not necessarily if the line is small enough).
Using this data, the Shell can roughly estimate the width of any line given the
number of bytes in it.

## Syntax Highlighting

The data structure of the index should enable efficient (and possibly lazy)
updates. Laziness is useful when stale entries may be used to speculate the
actual values (which is often the case).

Four useful fields per index entry:

- Offset: Byte offset of this entry from the beginning of file
- Version: Version of the buffer using which this entry was created
- State: The state of the parser
- Data: Highlight group

This part is very open ended and really depends on the type of parser used.
Furthermore, this component can be swapped with a language-specific semantic
checker to provide very rich highlighting (e.g. highlight errors, use symbol
table to determine the type of an identifier, etc.).

## Plugins

All plugins should be implemented on top of the Shell. Plugins are run on a
different process, and use IPC for all communication. Provide a wrapper API and
a "plugin host" for popular languages such as Python so that simpler plugins
which extends only a few aspects of the shell can be easily implemented (similar
to Sublime Text 3). Plugins can also be independent processes that need to be
spawned by the Shell, or are listening on a TCP address.

To help discover plugins, keep a centralized index of plugins. Dependencies
should possibly be defined using Capabilities (which are also indexed).

## Capabilities

A capability defines plugin APIs for certain tasks. A plugin may be a provider
of zero or more capabilities. Thus capabilities are a layer through which
plugins may interact and compose with each other.

## API

### Buffer

### The Shell

### The Skin

### The Filer

## Other design articles

- [GNU Emacs Internals](https://www.gnu.org/software/emacs/manual/html_node/elisp/GNU-Emacs-Internals.html#GNU-Emacs-Internals)
- [Architecture Analysis and Repair of Open Source Software (page 7 onwards)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.46.6702&rep=rep1&type=pdf)
- [Vim, an open-source editor (section 'Storing text' onwards)](http://www.free-soft.org/FSM/english/issue01/vim.html)
- [Xi Editor: Rope Science](https://github.com/google/xi-editor/blob/master/doc/rope_science/intro.md)
- [Kate internals: Text buffer](https://kate-editor.org/2010/03/03/kate-internals-text-buffer/)
- [Vis: Text management using a piece table](https://github.com/martanne/vis/blob/master/README.md#text-management-using-a-piece-tablechain)
