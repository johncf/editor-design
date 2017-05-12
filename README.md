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
- [The Cursor](#the-cursor)
  - [Vim-like implementation](#vim-like-implementation)
  - [Text Objects and Cursor motion](#text-objects-and-cursor-motion)
- [Regular Expressions](#regular-expressions)
- [Performance Optimizations](#performance-optimizations)
  - [Scroll Bars](#scroll-bars)
  - [Syntax highlighting](#syntax-highlighting)
- [Features of Shell](#features-of-shell)
  - [Plugins](#plugins)
  - [Capabilities](#capabilities)
  - [Dummy Lines](#dummy-lines)
- [Other design articles](#other-design-articles)

## Introduction

This was born out of the fascination I have towards text editors and see how I
would design one, borrowing many great ideas from our vast universe. The focus
was on making the whole thing as asynchronous and non-blocking as possible, and
consequently allowing stuff to be run in parallel.

This repository is [dedicated to the public domain (CC0)](LICENSE).

[design-articles]: #other-design-articles

## Design Goals

- Reliable: don't lose stuff
- Responsive: non-blocking
- Modular: independent modules with clean interfaces
- Extensible: where the fun begins!

## Non-goals

- Define a new scripting language
- Support for proportional font rendering

## Terminology

- Buffer: Represents the contents of the file being edited.
- Core: The component which handles raw text and file operations.
- Shell: The user interface.

## Architecture overview

The editor has two main components: Core and Shell.

![Architecture diagram](https://johncf.github.io/editor-design/arch.svg)

### Concerns of Core

- [Buffer management](#buffer-management)
- Handle text editing operations: insert, delete
- Handle history operations: undo, redo
- File IO: concerning the actual file, and its swap and undo files
- Serve buffer contents in chunks on-demand.
  - When the shell requests for a range of lines, it may specify a maxlength,
    to avoid fetching huge lines at once.
- Serving buffer summaries such as number of lines, byte-count per line

### Concerns of Shell

- Typesetting and rendering
- Cursor movements, text selection, and scrolling
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
editor. The following design was heavily influenced by [the CRDT model][].

The text data, along with the history of changes will be captured by the
following four entities:

- _A versioned Text Wall,_ which is an ever-growing (never-shrinking) sequence
  of characters, where any text insertions are inserted into Text Wall
  "in-context" and text deletions are ignored. This is probably best represented
  by a [persistent rope-like data structure][rope-wiki].
- _A strictly linear history of Text Wall,_ which sequence of revision ids and
  the corresponding interval(s) of inserted text. This will be used for
  computing the index position w.r.t the latest revision of Text Wall, given an
  index w.r.t a past version of the wall.
- _A mask on Text Wall,_ which represents the actual buffer contents. A mask is
  tied to a specific revision of Text Wall, and contains a sequence of intervals
  which "masks" Text Wall to obtain the buffer contents at that point.
- _An undo-tree of actions on the buffer,_ where an action is a set of `show`-s
  and `hide`-s.  A `show` or a `hide` is a change brought to the previous
  version of the mask.

[the CRDT model]: https://github.com/google/xi-editor/blob/6683f20/doc/crdt.md
[rope-wiki]: https://en.wikipedia.org/wiki/Rope_(data_structure)

### Handling huge files using `mmap`-ed buffers

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

![Cursor movements](https://johncf.github.io/editor-design/cursor.svg)

Note: `d`-commands are not affected by the position of cursor block.

## Regular Expressions

The Shell will provide two simple mechanisms involving regular expressions:

- Search
- Search and replace
- Search and replay (a macro)

Apart from these, the user must be encouraged to use external tools such as
`awk` or `sed`. To that end, the user may be provided with a feature to
"preview" such commands. A "preview" is just a partial view the processed
result in order to minimize execution time while editing the command.

## Performance Optimizations

Certain details about the buffer may be indexed (cached) by the Shell to boost
performance. These should be implemented only if the performance can be
provably improved.

The data structure of the index should enable efficient (and possibly lazy)
updates. Laziness is useful when stale entries may be used to speculate the
actual values (which is often the case).

### Scroll Bars

The baseline scrolling may be behave as described below:

- Vertical scroll-bars need only reflect the number of lines, even if
  text-wrapping is enabled.
- Mouse-wheel scrolling, cursor movements, or scroll-bar arrow-keys may be used
  for finer control.
- Scroll-bar thumb may be disabled when handling huge files with just a single
  line (or too few lines).

For smooth scrolling, the scroll-height value must be based on the actual
number of rows needed to render the whole lines which is hard to calculate when
text-wrapping is enabled. So caching this information might be useful. If the
file is too big, we may defer processing the entire file, and insted speculate
the number of rows needed based on the lines that were processed, and the
actual render widths may be calculated when the cursor approaches them. For
each line store:

    render_width, num_rows, window_width, buffer_version

Note: Tab characters may mess with render width speculations.

This cached list should not be updated every time something changes globally
(such as window-width or tab-width). This whole deal is just to get an
approximate scroll height. Once the list is constructed, it should only be
updated when text is actually rendered (at which point, adjust the
scroll-height too if necessary).

### Syntax Highlighting

Four useful fields per cache entry:

- Offset: Byte offset of this entry from the beginning of file
- Version: Version of the buffer using which this entry was created
- State: The state of the parser
- Data: Highlight group

This part is very open ended and really depends on the type of parser used.
Furthermore, this component can be swapped with a language-specific semantic
checker to provide very rich highlighting (e.g. highlight errors, using symbol
table to determine the type of an identifier, etc.).

## Features of Shell

### Plugins

All plugins should be implemented on top of the Shell. Plugins are run on a
different process, and use IPC for all communication. Provide a wrapper API and
a "plugin host" for popular languages such as Python so that simpler plugins
which extends only a few aspects of the shell can be easily implemented (similar
to Sublime Text 3). Plugins can also be independent processes that need to be
spawned by the Shell, or are listening on a TCP address.

To help discover plugins, keep a centralized index of plugins. Dependencies
should possibly be defined using Capabilities (which are also indexed).

### Capabilities

A capability defines plugin APIs for certain tasks. A plugin may be a provider
of zero or more capabilities. Thus capabilities are a layer through which
plugins may interact and compose with each other.

### Dummy Lines

These are lines which are inserted between the buffer contents but are
completely managed by (a plugin via) the Shell. These lines are not editable by
the user. This feature may be used by diff-plugins, compiler-plugins, and the
like. This can be extended to Dummy Buffers which completely bypasses the Core.

## Other design articles

- [GNU Emacs Internals](https://www.gnu.org/software/emacs/manual/html_node/elisp/GNU-Emacs-Internals.html#GNU-Emacs-Internals)
- [Architecture Analysis and Repair of Open Source Software (page 7 onwards)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.46.6702&rep=rep1&type=pdf)
- [Vim, an open-source editor (section 'Storing text' onwards)](http://www.free-soft.org/FSM/english/issue01/vim.html)
- [Kate internals: Text buffer](https://kate-editor.org/2010/03/03/kate-internals-text-buffer/)
- [Vis: Text management using a piece table](https://github.com/martanne/vis/blob/master/README.md#text-management-using-a-piece-tablechain)
