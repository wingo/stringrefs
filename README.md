# Host-Managed Strings in WebAssembly

A minimal-viable-product (MVP) proposal for WebAssembly.

## Champions

Andy Wingo `<wingo@igalia.com>`

## Goals

 1. Enable C++/WebAssembly programs to efficiently create and consume
    JavaScript strings
 2. Provide a good string implementation that many languages implemented
    on top of the GC proposal would find useful

These goals are sometimes in tension!  The operative words to help us
find good compromises are "minimal" and "viable".

## Requirements

 1. Zero-copy passing of strings from JavaScript to WebAssembly & back
 2. No new string implementations on the web: allow re-use of JS
    engine's strings
 3. Allow UTF-8-only implementation for non-web WebAssembly
    implementations
 4. Allow string literals in element sections

## Definitions
 - *codepoint*: An integer in the range [0,0x10FFFF].
 - *surrogate*: A codepoint in the range [0xD800,0xDFFF].
 - *unicode scalar value*: A codepoint that is not a surrogate.
 - *character*: An imprecise concept that we try to avoid in this
    document.
 - *code unit*: An indivisible unit of an encoded unicode scalar value.
    For UTF-8 encodings, an integer in the range [0,0xFF] (a byte); for
    UTF-16 encodings, an integer in the range [0,0xFFFF]; for UTF-32,
    the unicode scalar value itself.
 - *high surrogate*: A surrogate in the range [0xD800,0xDBFF].
 - *low surrogate*: A surrogate which is not a high surrogate.
 - *surrogate pair*: A sequence of a *high surrogate* followed by a *low
    surrogate*, used by UTF-16 to encode a codepoint in the range
    [0x10000,0x10FFFF].
 - *isolated surrogate*: Any surrogate which is not part of a surrogate
    pair.

## Design

### Implications of using JavaScript engine string representations

The requirement to re-use JavaScript string representations has a number
of implications.

#### Immutable strings

JS strings are immutable, so WebAssembly strings should also be
immutable.

#### Polymorphism

JS engines typically represent strings in many different ways: strings
which are "narrow" (in which all code units are in [0,0xFF] and can use
a fixed-width encoding with only one byte per code unit) or "wide" (two
bytes per code unit, 1 or 2 code units per codepoint), rope strings or
not, string slices or not, and external or not.  That's at least 16
different kinds of strings.

JavaScript can mitigate this polymorphism to a degree via adaptive
compilation, which can devirtualize based on the kind of strings seen at
the use site.  However, as of late 2021, no WebAssembly implementation
has needed to reach to adaptive compilation to get performance; we
shouldn't design a string interface that requires JIT heroism to be
fast.

At the same time, when looping over codepoints in a string, you would
really like your WebAssembly to inline its accessors and iterators.
Therefore we will lean away from interfaces that encourage one or more
string instruction per character, and instead will encourage encoding
(sub-)strings to UTF-8 in memory and operating over those chunks.

### JavaScript interoperability and surrogates

The WebAssembly strings proposal has to consider surrogates because we
want WebAssembly strings to have good interoperation with JavaScript,
whose strings are composed of 16-bit code units, not codepoints and not
unicode scalar values.

The code units of a JavaScript string are usually interpreted as
UTF-16-encoded unicode scalar values.  Some unicode scalar values are
encoded as one code unit, and some as two-code-unit surrogate pairs.

When [encoding a JavaScript string to
UTF-8](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder/encode),
logically one first decodes the sequence of code units to a
[USVString](https://developer.mozilla.org/en-US/docs/Web/API/USVString),
then encodes those USVs as UTF-8.  However, the first decoding phase can
fail for isolated surrogates.  In JavaScript, isolated surrogates can
occur via:
  - Reading invalid UTF-16 from external sources.  However this is not
    common, as isolated surrogates are not valid UTF-16.
  - JavaScript code that creates strings whose code units are not valid
    UTF-16.
  - JavaScript code that processes strings in chunks and happens to
    split a chunk on a surrogate boundary.
    - This happens most often in JavaScript code that processes strings
      one code unit at a time.
  - [JavaScript / DOM keyboard input event
    handlers](https://github.com/rustwasm/wasm-bindgen/issues/1348#issuecomment-473186426)
    (though this may be just a bug;
    [[1]](https://bugzilla.mozilla.org/show_bug.cgi?id=1541349),
    [[2]](https://bugs.chromium.org/p/chromium/issues/detail?id=949056)).

Isolated surrogates are a source of bugs when programs need to produce
standard Unicode encodings.  Programs can generally handle them in three
ways:
 - Some programs replace isolated surrogates with the unicode
   replacement character (U+FFFD), losing information but allowing
   processing to proceed.
 - Other programs raise an exception, stopping the program unexpectedly.
 - Others attempt to preserve information by using non-standard
   encodings.  For example, a program can encode to
   [CESU-8](https://en.wikipedia.org/wiki/CESU-8) or
   [WTF-8](https://simonsapin.github.io/wtf-8/) instead of UTF-8.

Though our primary motivation for this proposal relates to JavaScript,
the concerns about surrogates also apply to some other common languages,
notably C# and Java.

If we just considered JavaScript, the most natural thing for WebAssembly
to do would be to consider strings as being composed of 16-bit code
units.  However this does not map to many other languages.

If we were just considering simplicity, the best solution would be to
say "strings are sequences of unicode scalar values", but we know that
for JavaScript this is not the case.

However, we think we can get closer to the simple solution by primarily
working in terms of unicode scalar values.  WebAssembly on its own
should not be able to create strings with isolated surrogates, and
therefore we should only include support for reading and writing
standard Unicode encoding schemes which exclude isolated surrogates by
construction, for example UTF-8 and UTF-16.  Isolated surrogates are
rare in JavaScript and the tail should not wag the dog.

Such problematic strings can come from a host, however, and where it is
as simple to define a behavior as to require an implementation to trap,
we will lean towards defined non-trapping behavior.  The proposal also
leaves the door open to add interfaces that access string contents using
more general encoding schemes such as WTF-8 if needed in the future.

#### No `get-char-at` method

A JavaScript string is composed of a sequence of 16-bit code units which
encode a sequence of codepoints, in which each codepoint corresponds to
1 or 2 code units.  In the general case, getting the *n*th unicode
scalar value from a string requires parsing all preceding code units.
We would not want to design an API that would encourage straightforward
uses (e.g. looping over unicode scalar values) to run in quadratic time.

### Oracle: JS engine C++ API

While developing this proposal, we realized that we already had a design
oracle: `v8.h`.  It is reasonable to think that the C++ interface to a
JS engine already has the minimal necessary set of interfaces.
 - We can assume that `v8.h` has all the interfaces that Chromium needs,
   so we expect that the interfaces in `v8.h` are sufficient.
 - V8 wants to minimize API surface and historically has removed API, so
   we expect V8's interface is close to minimal.
 - C++ interfaces to different JS engines are similar.  We can look at
   `v8.h` and draw conclusions for any engine.

The [V8 C++ String
API](https://chromium.googlesource.com/v8/v8/+/refs/heads/main/include/v8-primitive.h#110)
includes the following procedural interfaces:

  - Create a string from encoded bytes in memory
    - Supported encodings: `one-byte`, `utf-8`, `utf-16`
  - Get length of string when encoded as `one-byte`, `utf-8`, `utf-16`
    - Does not include unicode scalar value count
  - Predicate on string to identify strings represented using one byte
    per character (a cheap check) and strings that can be represented
    using one byte per character (possibly a linear search)
  - Write encoded bytes to memory
    - Supported encodings: `one-byte`, `utf-8`, `utf-16`
    - Options: hint that ropes should be flattened, include `NUL`
      terminator or not, whether to preserve `NUL` codepoints, whether
      to replace isolated surrogates with the replacement character or
      to trap
  - Support for strings whose characters are in linear memory and which
    shouldn't be copied ("external strings"); probably not appropriate
    for WebAssembly
  - Equality predicate
  - Concatenate two strings.  A complexity hazard: encodes performance
    characteristics of rope strings into API

### Prior discussions

 * The component model subgroup chose to agree that on component
   boundaries, strings consist of sequences of USVs:
   https://docs.google.com/presentation/d/1qVbBsDFmremBGVKiOAzRk7svjinNq6LXfJ1DzeFwKtc
   *The CG discussion and decision inform but don't constrain this proposal.*

 * The AssemblyScript developers floated a "universal strings" proposal
   which explicitly provided for the WTF-8 and WTF-16 encodings that can
   support unpaired surrogates:
   https://github.com/AssemblyScript/universal-strings
   *A quite workable solution if we determine that it's necessary to
   support unpaired surrogates.  Can we preserve the possibility to add
   support for WTF-8 if needed, while not requiring it in an MVP?*

## Proposal

 1. Only provide interfaces to create strings from encoded bytes in
    memory.
 2. Provide for three in-memory encodings of USV sequences: `one-byte`,
    for values in [0,255], and `utf-8` and `utf-16` (little-endian), for
    any valid unicode scalar value.
 3. Only provide interfaces to access string contents via writing
    encoded bytes to memory.  Use an iterator to avoid eager flattening
    of rope strings.  Support the ability to know how many bytes the
    encoded data will take, to support precise allocations, to know when
    the `one-byte` encoding can be used, and to know if a string
    originating in JavaScript has unpaired surrogates.
 4. Don't provide a `get-usv-at-index` accessor, to avoid O(n) search
    for `utf-8` and `utf-16` backing stores.
 5. When attempting to create a string from an invalid byte string or
    when encoding bytes from a string fails, trap.  If this proves too
    much of a constraint in practice, our future escape hatch is to add
    additional encodings.
 6. Don't provide a `concat` / `append` interface.  Because JS engines
    would implement this via rope strings, this would encode an
    expectation that rope strings are necessary.  In the MVP this
    operation can be supported by calling out to a JS import.

## Instruction set

One new reference type: `stringref`.  Opaque, like `externref` and
`funcref`.

Many of these instructions treat string contents as being encoded in a
given encoding.  Those instructions take an immediate uleb *encoding*
parameter.

```
encoding ::= utf-8 | utf-16 | one-byte
```

When reading or writing encoded bytes, the address in memory at which to
read or write the bytes depends on the memory model of the WebAssembly
module.
```
address ::= i32 | i64
```
Such instructions also take the memory to which to read or write as an
immediate.

### Creating strings

```
(string.new $memory $encoding ptr:address len:i32)
  -> str:stringref
```
Create a new string from the *`len`* bytes encoded in memory at *`ptr`*.
Out-of-bounds access will trap.  The `utf-16` encoding implies two-byte
alignment for both operands and will trap otherwise.  Attempting to
create a string with bytes that are not valid for the encoding will
trap.  No terminating `NUL` byte is expected.

#### `string.new` size limits

The maximum value for *`len`* is 2<sup>31</sup>–1 bytes.  Passing a
higher value traps.

Creating a string is a form of dynamic allocation and can fail.  The
same implementation running on different machines can have different
behaviors.  The specification can only say that byte sizes above a
certain limit *must* fail; but for byte sizes within the limits, the
allocations *may* fail.  If an allocation fails, the implementation
must trap.  Fallible `string.new` is a possible future extension.

### String literals

```
(string.const contents:i32)
  -> str:stringref
```
Create a new string from the literal string *`contents`*, as in
`(string.const "Hello, World!")`.  This instruction is constant and can
be used in global variable initializers.

#### String literal section

The `string.const` section indicates the literal as an `i32` index into
a new custom section: a string table, encoded as a `vec(vec(u8))` of
valid UTF-8 strings.  Because literal strings can contain codepoint 0,
strings in the string table do not use NUL as a terminator. The string table section must
immediately precede the global section, or where the global section
would be, in the binary.

#### `string.const` size limits

The maximum size for the UTF-8 encoding of an individual string literal
is 2<sup>31</sup>–1 bytes.  Embeddings may impose their own limits which
are more restricted.  But similarly to `string.new`, instantiating a
module with string literals may fail due to lack of memory resources,
even if the string size is formally within the limits.  However
`string.const` itself never traps when passed a valid literal offset.

### String cursors

To represent positions in strings for the purposes of accessing string
contents, we introduce the concept of "cursor", encoded as an `i32`.  A
string consisting of *N* unicode scalar values will have *N*+1 distinct
valid cursor values: one before each USV, and one at the end.  A cursor
value of 0 denotes the string start (before the first USV).  It follows
that 0 may also denote the end of the string also, for zero-length
strings.

A cursor value is an offset into a string.  Cursors uniquely identify a
position in a string, and are ordered, and therefore can be compared
against each other.

If a host represents strings internally using UTF-8, UTF-16, or UTF-32,
or a variant thereof such as the one used in JavaScript, cursor values
are code unit offsets.

For example, because JavaScript hosts have to represent strings as
(logical) sequences of 16-bit code units, a WebAssembly string cursor
for a WebAssembly implementation embedded in a web browser will be a
code unit offset (the same as the operand to JavaScript's
[`String.prototype.charCodeAt`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/charCodeAt).

For a WebAssembly implementation that represents strings as UTF-8
internally, cursor values are byte offsets.  The intention is that
accessing content in a string with a cursor has the least possible
overhead.  This also allows hosts to communicate string positions with
WebAssembly programs.

If a host does not represent strings using a unicode encoding scheme,
the specific mapping from USV offset to cursor value is
implementation-defined, with the requirement that values be ordered and
that each position must have one and only one cursor value.

To move a cursor to a new position, use the `string.advance` and
`string.rewind` seek instructions.

```
(string.advance str:stringref cursor:i32 codepoints:i32)
  -> cursor:i32, codepoints:i32
(string.rewind str:stringref cursor:i32 codepoints:i32)
  -> cursor:i32, codepoints:i32
```
Return a new cursor into the string *`str`* which is *`codepoints`*
codepoints after or before *`cursor`*.  Also return the number of
codepoints by which the new cursor differs from the old, which may be
less than the *`codepoints`* parameter if end-of-string or
beginning-of-string would be reached.  The *`codepoints`* argument is
interpreted as a `u32`.

#### Invalid cursor values

All instructions which take cursors trap if the cursor is not valid for
the given string.

#### Host strings which are not valid USV sequences

In this MVP, it is not possible to produce a string which is not a valid
sequence of unicode scalar values.

However, strings provided by some hosts may support invalid USV
sequences.  Notably, JavaScript, C#, and Java strings can include
isolated surrogates, which are not valid unicode scalar values.  If a
string created by a host contains isolated surrogates, there is a valid
cursor position before each isolated surrogate.  In this way,
`string.advance` and `string.rewind` do not trap for strings containing
isolated surrogates.

#### Cursor range limits

The maximum valid `i32` cursor value is 2<sup>31</sup>.  Processing a
string whose highest cursor value would be greater than 2<sup>31</sup>
with the `string.advance`, `string.rewind`, `string.cur`,
`string.measure`, `string.encode`, and `string.eq` instructions must
trap.

Given that there must be a distinct cursor value for each codepoint in a
string, and one for the end, this constrains the strings that are
processed by to a maximum of 2<sup>31</sup>–1 code units.  Not all
strings have one code unit per codepoint, so the codepoint size limit in
practice may be lower for any given string.  This specification can
therefore represent codepoint counts with an `i32` without risk of
overflow.

Future extensions of the string proposal along the lines of the
[memory64
proposal](https://github.com/WebAssembly/memory64/blob/main/proposals/memory64/Overview.md)
may allow for 64-bit variants of the cursor-using instructions, which
could relax these restrictions.

In practice, no web browser embedding allows for strings longer than
2<sup>31</sup>–1 UTF-16 code units, so no string from JavaScript is too
large for the instructions in this proposal.

### Accessing string contents

All parameters and return values measuring a number of codepoints or a
number of bytes represent these sizes as unsigned values.

```
(string.cur str:stringref cursor:i32)
  -> codepoint:i32
```
Return the codepoint at *`cursor`* in *`str`*, as an i32.  The result is
a unicode scalar value, unless the host's strings support isolated
surrogates, in which case the result may be a surrogate.  If *`cursor`*
is at the end of *`str`*, the result is -1.

```
(string.measure $encoding str:stringref cursor:i32 codepoints:i32)
  -> bytes:i32
```
Measure how many bytes would be necessary to encode the contents of the
string *`str`*, starting at cursor *`cursor`*, limited to a maximum of
*`codepoints`* codepoints.  If the contents of the string cannot be
encoded in the given encoding, return -1.

The maximum number of bytes returned by `string.measure` is
2<sup>31</sup>-1.  If an encoding would require more bytes, it is as if
the contents can't be encoded at all (the return value is -1).

(Note that the limit on cursor values constrains strings to have a
maximum of 2<sup>31</sup>–1 codepoints, but it may be that such a string
could exceed even 2<sup>32</sup> bytes in a given encoding; e.g. a
string of 2<sup>31</sup>-1 `\uFFFF` codepoints would require
2<sup>32</sup>+2<sup>31</sup>-3 bytes when encoded as UTF-8.)

```
(string.encode $encoding $memory str:stringref cursor:i32 ptr:address bytes:i32)
  -> cursor:i32, codepoints:i32, bytes:i32
```
Encode the contents of the string *`str`*, starting at the cursor
*`cursor`*, to memory at *`ptr`*, limited to *`bytes`* bytes.  Return
three values: a string cursor pointing to the remaining contents of the
string, the number of codepoints written, and the number of bytes
written.  Note that no `NUL` terminator is ever written.  If any
codepoint cannot be encoded, trap.

The maximum number of bytes that can be encoded at once by
`string.encode` is 2<sup>31</sup>-1.  If an encoding would require more
bytes, it is as if the codepoints can't be encoded (a trap).

### Predicates

```
(string.eq a:stringref a-cursor:i32 b:stringref b-cursor:i32 codepoints:i32) -> i32
```
Return 1 if the strings *`a`* and *`b`* contain the same codepoint
sequence, starting at the cursors *`a-cursor`* and *`b-cursor`*
respectively, limited to comparing up to *`codepoints`* codepoints.
Return 0 otherwise.

## Examples

We assume that the textual syntax for `string.encode` and `string.new`
allows you to elide the memory, in which case it defaults to 0.

### Make string from NUL-terminated UTF-8 in memory

```wasm
(func $string-from-utf8 (param $ptr i32) (result stringref)
  local.get $ptr
  local.get $ptr
  call $strlen
  string.new utf-8)
```

### Make string from one-byte codepoints in memory

```wasm
(func $string-from-latin1 (param $ptr i32) (param $len i32) (result stringref)
  local.get $ptr
  local.get $len
  string.new one-byte)
```

### Make string from UTF-16 in memory

```wasm
(func $string-from-utf16 (param $ptr i32) (param $units i32) (result stringref)
  local.get $ptr
  local.get $units
  i32.const 1
  i32.shl ;; code units to bytes
  string.new utf-16)
```

### Number of codepoints in string

```wasm
(func $codepoint-length (param $str stringref) (result i32)
  local.get $str
  i32.const 0    ;; initial cursor: beginning of string
  i32.const -1   ;; advance by all codepoints
  string.advance ;; push new cursor, codepoints
  return)        ;; just return codepoints
```

### String literals

```wasm
(global $hey stringref (string.const "Hey"))

(func $howdy (result stringref)
  (string.const "Howdy"))

(func $is-cowboy (param $str stringref) (result i32)
  local.get $str
  i32.const 0
  call $howdy
  i32.const 0
  i32.const -1
  string.eq)
```

### Suffix, prefix comparisons

```wasm
(func starts-with-hey? (param $str stringref) (result i32)
  global.get $hey
  i32.const 0
  local.get $str
  i32.const 0
  i32.const 3
  string.eq)

(func ends-with-howdy? (param $str stringref) (result i32)
  string.const "Howdy"
  i32.const 0
  local.get $str

  ;; Get cursor 5 codepoints before end of $str:
  local.get $str
  ;; First get cursor at end of $str...
  local.get $str
  i32.const 0
  i32.const -1
  string.advance
  drop
  ;; ...then rewind by 5.
  i32.const 5
  string.rewind

  ;; Limit comparison to 5 codepoints.
  i32.const 5
  string.eq)
```

### Store a stringref without copying

```wasm
(table $strings 100 stringref)
(global $next-handle i32 (i32.const 0))

(func $intern-string (param $str stringref) (result i32)
  (local $handle i32)
  global.get $next-handle
  local.tee $handle
  local.get $str
  table.set $strings
  i32.const 1
  i32.add
  global.set $next-handle
  local.get $handle)
```

### Copy string contents to application-managed memory

```wasm
(func $malloc (param i32) (result i32))
(func $utf8-contents (param $str stringref) (result i32)
  (local $cur i32)
  (local $len i32)
  (local $ptr i32)
  local.get $str
  i32.const 0
  i32.const -1
  string.measure utf-8
  local.set $len

  block $valid
    local.get $len
    i32.const -1
    i32.ne
    br_if $valid
    unreachable                    ;; trap on error
  end

  local.get $len
  i32.const 1
  i32.add
  call $malloc                     ;; reserve space for bytes and NUL
  local.set $ptr

  local.get $str
  i32.const 0
  local.get $ptr
  local.get $len
  string.encode utf-8              ;; push cursor, codepoints, and $len

  local.get $ptr
  i32.add
  i32.const 0
  i32.store8                       ;; write NUL

  local.get $ptr
  return)
```

### Stream over contents of string

Assume you have a 1024-byte array of memory at `$buf`.  This function
will trap on isolated surrogates.

```wasm
(global $buf i32)
(func $process-utf8 (param $ptr i32) (param $len i32))

(func $process-string (param $str stringref)
  (local $cursor i32)                ;; initial value of 0 is start
  (local $bytes i32)

  loop
    local.get $str
    local.get $cursor
    global.get $buf
    i32.const 1024
    string.encode utf-8              ;; push new cursor, codepoints and $len
    local.tee $bytes
    (if i32.eqz (then return))       ;; if no bytes encoded, done
    drop ;; ignore codepoints
    local.set $cursor

    global.get $buf
    local.get $bytes
    call $process-utf8
  end)
```

### Stream over codepoints of string, handling isolated surrogates

This function is probably slower than handling strings in chunks.

```wasm
(func $have-codepoint (param $codepoint i32))

(func $process-string (param $str stringref)
  (local $cur i32)
  (local $ch i32)

  block $done
    loop $loop
      local.get $str
      local.get $cur
      string.cur
      local.set $ch

      local.get $ch
      i32.const -1
      i32.eq
      br_if $done

      local.get $ch
      call $have-codepoint

      local.get $str
      local.get $cur
      i32.const 1
      string.advance
      drop
      local.set $cur
    end
  end)
```

## FAQ

### What does Emscripten currently do for strings and how could WebAssembly strings help?

Generally speaking, Emscripten [eagerly converts JavaScript strings to
NUL-terminated
UTF-8](https://github.com/emscripten-core/emscripten/blob/main/src/preamble.js#L91-L100),
allocating space for the UTF-8 encoding in linear memory using stack
allocation.  The [stringToUTF8
function](https://github.com/emscripten-core/emscripten/blob/main/src/runtime_strings.js#L158)
is written in JavaScript and handles surrogate pairs.  However for
isolated surrogates, [emscripten's decoder appears to produce
garbage](https://github.com/emscripten-core/emscripten/issues/15324).

For C functions that return strings, emscripten [parses NUL-terminated
UTF-8 from
memory](https://github.com/emscripten-core/emscripten/blob/main/src/preamble.js#L109),
[either using TextDecoder or via hand-rolled
JavaScript](https://github.com/emscripten-core/emscripten/blob/main/src/runtime_strings.js#L35).
Presumably TextDecoder is significantly faster as it doesn't have to
build rope strings.

Memory management is an issue, of course; the memory for a returned
string value may or may not be owned by the caller.

This proposal avoids memory ownership issues entirely, via automatic
memory management (implemented either via GC or reference counting).  It
also avoids eager string encoding onto the stack and the need for NUL
termination, allowing string contents to be written to memory exactly
where they are needed.

https://github.com/guybedford/proposal-is-usv-string

### What is the expected implementation on non-browser run-times?

Assuming that the non-browser implementation uses UTF-8 as the native
string representation, then a stringref is a pointer, a length, and a
reference count.  The specification requires that cursor values be UTF-8
code unit offsets, which are byte offsets from the beginning of the
string.  Cursor validation is ensuring the cursor is less than or equal
to the string byte length, and that `(ptr[cursor] & 0xb0) != 0x80`.
Measuring UTF-8 encodings is just length minus the cursor.  Measuring
UTF-16 would be via a callout.  Encoding UTF-8 is just `memcpy`.

### What's the expected implementation in web browsers?

We expect that web browsers use JS strings as `stringref`.  The
specification then requires that cursor values be UTF-16 code unit
offsets.  Seeking, measuring, encoding, and equality predicates would
likely call out to run-time functions that would dispatch over
polymorphic values.  The support for one-byte encodings may prove to be
a performance benefit also.

### Why define string cursors in terms of the host's string representation?

The purpose of a string cursor is to allow efficient access to string
contents, starting at a specific position.

Under the hood, string cursors must relate to host string
representation.  For example, we really want to support efficient access
to JavaScript strings, so string cursors in a web browser should express
positions in terms of UTF-16 code unit offsets.  But we don't want
WebAssembly strings to be specified in terms of UTF-16 only; non-web
embeddings will likely represent strings internally using other
encodings (often UTF-8).  So instead we advance cursors in units of
unicode scalar values, with some allowances for isolated surrogates from
JavaScript.  But we can't define string cursors as being USV offsets,
because mapping USV offset to code unit offset is O(n).  Cursors allow
us to avoid quadratic algorithms.

The question then becomes, because cursor values relate to a host's
string representation, should we hide the details of what a string
cursor is from users, in the name of abstraction and common defined
behavior?

All things being equal, it would have been nice to define string cursors
in such a way that a program running on a UTF-8 host would behave
exactly the same as for a UTF-16 host.  We could have provided this
property by making string cursors opaque.  This could have gone two
ways: if we made cursors a first-class reference-typed value, cursors
could hold a reference to their strings directly.  There would then be
no need for cursor validity checks.  On the other hand, then we would
have a new type that would infiltrate everything, from implementation to
JavaScript API to the type system and so on.  And, absent compiler
heroics, reference-typed cursors may cause high allocation overheads.

The other way you could make cursors opaque would be as opaque scalar
values.  The idea is that a cursor is really an `i32` under the hood,
but its value isn't accessible.  Such a cursor wouldn't stand alone in
the way reference-typed cursors would: you need to pass a string and a
cursor to instructions, and you need to check the cursor for validity
with regards to the string.  We still have some of the type profusion
issues from a "cognitive load" point of view.  But, you couldn't observe
the difference in cursor values between implementations, which would be
a nice property.

In the end though, besides simplicity, what tipped the balance towards
plain `i32` values was precisely that string cursors could be meaningful
to the host instead of opaque.  A host should be able to reason about
string positions and communicate those positions to WebAssembly -- after
all, the strings belong to the host too.  Specifying that string cursors
are code unit offsets makes this possible, while also constraining
e.g. WebAssembly implementations in different web browsers to all use
the same notion of string offsets.

See https://github.com/wingo/wasm-strings/issues/6 and
https://github.com/wingo/wasm-strings/issues/11 for a full discussion.

### Are stringrefs nullable?

Oh God I guess so.  `ref.null string` it is I guess!!  :sob: :sob: :sob:

### What kinds of performance gains can we expect?

 * WebAssembly can receive encoded content of JS strings exactly where
   it is wanted: no need to stack-allocate then copy.
 * WebAssembly can process long strings in chunks rather than having to
   reserve space for the whole string.
 * WebAssembly can cheaply check incoming strings against literals,
   treating them as symbols.
 * Avoid JIT warmup for JS-implemented UTF-8 encode and decode.
 * Avoid allocation of subarrays when decoding; e.g. [as used by
   emscripten](https://github.com/emscripten-core/emscripten/blob/da842597941f425e92df0b902d3af53f1bcc2713/src/runtime_strings.js#L48)
 * Cheap prefix/suffix tests without reading whole string
 * WebAssembly can cheaply pass string literals to JS without decoding
   or copying

### What's the security implication?

Right now, working with strings fundamentally means communicating UTF-8
via memory.  To grant someone access to a string, you have to grant them
access to all of your memory.  This violates the principle of least
privilege.  Having reference-typed strings would limit the capability to
just the immutable codepoint sequence in question, and not all of
memory.

Additionaly, interfacing between memory lifetimes in C/C++ and
JavaScript is bug-prone.  WebAssembly strings would eliminate questions
of memory ownership, reducing the risk of use-after-free, data
corruption, write overruns, and privileged data leakage.

### Doesn't the `encode` interface imply some copying overhead?

It's true that it would be nice to read the contents of a string
"directly".  However given the polymorphism of JS strings, this doesn't
seem to be possible in theory; and as strings are reference-typed,
there's no interface currently to be able to read and write their
contents.  `i32.load8_u` only works from memory, not GC-managed objects.
That said though, any copy is likely to remain in cache, amortizing the
cost of the second access.  Inlining the (likely) UTF-8 accesses on the
WebAssembly side seems more important than preventing a copy by using a
codepoint-by-codepoint non-copying interface.

### Why not just use `externref` and imported functions?

The instruction set could be implemented with imported functions,
replacing the `stringref` type with `externref`.  So why bother adding
it to WebAssembly itself?  Three reasons: platform expressivity,
performance, and security.

On the first point: if the strings feature required some capability from
the host, then it would be clearly best as a library.  For example,
WebGL access falls in this category.  But reference-typed strings are a
more fundamental feature common to all languages that use automatic
memory management.  In that way they are closer to the GC proposal;
although you could implement structs and arrays via externref and
imports, if you did that you might as well compile to JavaScript instead
of WebAssembly.  It should be possible to make a WebAssembly program
that uses reference-typed strings (because almost all such programs
would have strings) without relying on any JavaScript at all.

Also, the evolutionary endpoint of an externref-and-imports strategy is
a JavaScript-specific string interface.  Without any broader WebAssembly
platform concern, strings-using WebAssembly code would find itself
relying on details of JavaScript's string representation, for example
processing strings one code unit at a time instead of one codepoint at a
time.  This is not a good platform outcome.

Finally, though the WebAssembly platform should be able to stand alone,
it should also interoperate smoothly with hosts, especially JavaScript
on the web.  This rules out any implementation of strings in terms of
reference-typed structs and arrays: not only would such an
implementation be likely slower than the host's strings, it would also
be incompatible.  On the web, WebAssembly and JavaScript should use the
same string implementation.

On the performance side, we expect that WebAssembly strings will be
faster than externref+imports:

 1. Whereas externrefs might need to be a tagged union, stringrefs can be
    unpacked pointers.
 2. WebAssembly instructions are likely faster and less of an
    optimization barrier than callouts to imports.
 3. Run-time helper code for WebAssembly instructions is probably
    implemented in C++/Rust/etc more directly, resulting in more
    predictable performance than e.g. an encoder implemented in JS (for
    web embeddings).
 4. Reading string contents via
    `string.encode`-to-UTF-8-then-process-inline is likely
    faster than calling out to JavaScript to read code units one at a
    time.  WebAssembly-to-JavaScript calls are cheap but not free.

On the other hand, it's true that JS run-time routines can use adaptive
JIT techniques to possibly inline representation-specific accessors.
This is of limited use though for run-time routines with many different
call sites.

On the reliability and security side, adding strings to WebAssembly
removes a significant user of extra-module access to memory.  Because
the WebAssembly code can pick apart the string itself, that's one fewer
reason for the WebAssembly module to have to expose its memory.

### How does this relate to the component model and interface types?

The component model is [a vision of how to compose systems out of
shared-nothing parts implemented in
WebAssembly](https://github.com/WebAssembly/component-model/pull/1/files).
The boundaries between these components are mediated by interface types,
which specify how to communicate data from one component to another in
the most efficient way possible.

At one level, reference-typed strings don't appear have anything to do
with the component model.  Because components are specified to not share
anything, even GC-managed data, we don't expect WebAssembly strings to
pass across a component boundary in the zero-copy way that this proposal
has as a design goal.

From the perspective of the component model, WebAssembly strings are
rather an *intra-component* concern.  A component may be composed
internally of a number of WebAssembly modules, as well as possible host
facilities such as JavaScript.  The zero-copy properties provided by
WebAssembly strings apply to the inter-module, intra-component
boundaries of a program.

That said, strings in the abstract are an important data type, and
relate to interface types (a WebAssembly proposal based on the component
model).  Obviously you will want to be able to use WebAssembly strings
with interface types.  The shared-nothing design choice of the component
model then implies that string contents should be copied when they cross
a component boundary.

Incidentally, for inter-component interfaces that deal in strings, the
component model specifies that abstractly, [strings are sequences of
unicode scalar
values](https://github.com/WebAssembly/interface-types/issues/135).
This implies that some JavaScript strings can't traverse a component
boundary, because of the potential for isolated surrogates, and also
implies an eager check that a WebAssembly string is valid, for an
interface-typed call.  In practice this is not a problem because the
string's contents are being copied anyway and so can be validated at the
same time.

Interface types are used to specify a WebAssembly function's signature
in an abstract way.  This signature should then be compiled down to a
concrete adapter function specialized to the data representations used
by the caller and the callee.  The instruction set in this proposal can
be used to implement the adapter function for passing stringrefs as
strings; assuming that the adapter function is generated in such a way
that it has access to the target memory, `string.encode` can implement
the copy and validation at the same time.  `string.new` would be the
implementation of getting a stringref from an interface-typed string
value.

Note that because this proposal can't create strings that are not USV
sequences, a WebAssembly implementation embedded by a host that also
can't produce USV strings can avoid any component-model string
validation checks.  This would be the case notably for hosts that use
UTF-8 as their underlying string representation; passing a stringref to
an interface-type interface would probably compile down to just a memcpy
of the stringref's contents.
