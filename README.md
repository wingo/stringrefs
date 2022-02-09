# Reference-Typed Strings

A minimum viable proposal to add a reference-typed strings to
WebAssembly (the `stringref` proposal).

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
 4. Allow WTF-16 support for Java, Dart, Kotlin and similar languages
 5. Allow string literals in element sections

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

JS strings are immutable, so a WebAssembly `stringref` should also be
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

The `stringref` proposal has to consider surrogates because we want
WebAssembly strings to have good interoperation with JavaScript, whose
strings are composed of 16-bit code units, not codepoints and not
unicode scalar values.  Additionally we want WebAssembly strings to be a
good compilation target for Java and C# strings, which have similar
16-bit code-unit concerns as JavaScript.

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

If we just considered JavaScript, Java, and C#, the most natural thing
for WebAssembly to do would be to consider strings as being composed of
16-bit code units.  However this does not map to many other languages.

If we focused just on the kinds of strings that every language supports,
the best solution would be to say "a `stringref` is a sequence of
unicode scalar values", because we know that all languages can work with
such strings.  However this is just a subset of the kind of strings that
JavaScript, Java, and C# work with, so we can't be this simple.

Also on a practical level, Java and C# and so on will need to be able to
address UTF-16 code units, not just unicode scalar values.  Sometimes a
Java compiler will be able to treat a WebAssembly string as a sequence
of unicode scalar values but that would be a rare case.  Compare this to
a Scheme implementation, for example, which sees strings only as
sequences of unicode scalar values and doesn't provide a language-level
facility for addressing strings in terms of code units, apart from
explicit "encode this string to UTF-16 in a byte array" interfaces.

We can perhaps find our way out of this mess by noting that the 16-bit
code units in Java and JavaScript may be interpreted as a sequence of
codepoints in the [WTF-16](https://simonsapin.github.io/wtf-8/) encoding
-- so not just a sequence of unicode scalar values, but actually any
codepoint, including isolated surrogates.

Therefore this proposal defines WebAssembly strings as sequences of
codepoints, including isolated surrogates.  Encoding a WebAssembly
string to UTF-8 may fail, as UTF-8 is a proper Unicode encoding scheme
that excludes isolated surrogates.  Languages that choose to process
strings in UTF-8 chunks can decide what to do for these somewhat
anomalous strings -- use a replacement character, or abort.  This
proposal also allows encoding to the more general WTF-8 and WTF-16
encodings, for languages that can handle any codepoint sequence.  For
16-bit access we only provide the permissive WTF-16 encoding, as source
languages that want to address strings in terms of 16-bit code units are
generally tolerant to isolated surrogates.

We will provide the ability to denote string positions in WTF-8 or
WTF-16 offsets.  We provide interfaces to advance WTF-8 string positions
by units of codepoints (rather than code units), and assume that WTF-16
users prefer code units.  Whether to use WTF-8 or WTF-16 is more a
property of the source language; some compilers and language runtimes
will prefer one encoding or the other.

#### No `get-nth-codepoint` method

A JavaScript string is composed of a sequence of 16-bit code units which
encode a sequence of codepoints, in which each codepoint corresponds to
1 or 2 code units.  In the general case, getting the *n*th codepoint
from a string requires parsing all preceding code units.  We would not
want to design an API that would encourage straightforward uses
(e.g. looping over codepoints) to run in quadratic time.

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
  - Concatenate two strings.  Interestingly, `v8.h` has no interface to
    make a substring (slice).

### Prior discussions

 * The component model subgroup chose to agree that on component
   boundaries, strings consist of sequences of USVs:
   https://docs.google.com/presentation/d/1qVbBsDFmremBGVKiOAzRk7svjinNq6LXfJ1DzeFwKtc
   *The CG discussion and decision inform but don't constrain this proposal.*

 * The AssemblyScript developers floated a "universal strings" proposal
   which explicitly provided for the WTF-8 and WTF-16 encodings that can
   support unpaired surrogates:
   https://github.com/AssemblyScript/universal-strings
   *An excellent early draft which seriously tackles the WTF-16 problem.*

## Proposal

We strike a compromise between three goals:

 - Support WTF-16, for Java
 - Allow implementations to represent strings as WTF-8
 - Allow source languages to choose policy for invalid UTF-8

Concretely, we propose to:

 1. Provide interfaces to create strings from encoded bytes in memory.
 2. Provide three in-memory encodings of codepoint sequences: `utf-8`,
    `wtf-8`, and `wtf-16` (little-endian).  The `utf-8` encoding can
    only encode codepoints which are valid unicode scalar values; the
    other two encodings can encode any codepoint, including isolated
    surrogates.
 3. Provide interfaces to access string contents via writing encoded
    bytes to memory.  Avoid eager flattening of rope strings.  Support
    the ability to know how many bytes the encoded data will take, to
    support precise allocations.  Allow for encoding failure for the
    strict `utf-8` encoding.
 4. Denote positions in strings in code units with respect to particular
    encodings.  For WTF-8/UTF-8, this is bytes, and only some positions
    are valid.  For WTF-16, this is 16-bit code units, and any position
    within the string's bounds is valid.  Provide interface to advance a
    string position by a number of codepoints.  See discussion below.
 5. Provide a `get-wtf-16-code-unit-at-offset` primitive, to support
    languages whose strings are 16-bit code units, even for
    implementations that store strings as WTF-8.
 6. When attempting to create a string from an invalid code unit
    sequence or when encoding bytes from a string fails, trap.
 7. Provide `concatenate` and `slice` interfaces.  Note that for strings
    that are not valid unicode scalar value sequences, `concatenate` may
    join isolated surrogates.  `slice` takes two string positions as
    arguments, and thus exists in WTF-8/UTF-8 and WTF-16 variants.  The
    WTF-16 `slice` may create dangling surrogates.

## Instruction set

One new reference type: `stringref`.  Opaque, like `externref` and
`funcref`.

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
(string.new_wtf8 $memory ptr:address bytes:i32)
  -> str:stringref
```
Create a new string from the *`bytes`* WTF-8 bytes in memory at *`ptr`*.
Out-of-bounds access will trap.  Attempting to create a string with
invalid WTF-8 will trap.  The maximum value for *`bytes`* is
2<sup>31</sup>–1; passing a higher value traps.

```
(string.new_wtf16 $memory ptr:address codeunits:i32)
  -> str:stringref
```
Create a new string from the *`codeunits`* code units encoded in memory at
*`ptr`*.  Out-of-bounds access will trap.  *`ptr`* must be two-byte
aligned, and will trap otherwise.  The maximum value for *`codeunits`*
is 2<sup>30</sup>–1; passing a higher value traps.

#### `string.new` size limits

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
valid WTF-8 strings.  Because literal strings can contain codepoint 0,
strings in the string table do not use NUL as a terminator. The string
table section must immediately precede the global section, or where the
global section would be, in the binary.

#### `string.const` size limits

The maximum size for the WTF-8 encoding of an individual string literal
is 2<sup>31</sup>–1 bytes.  Embeddings may impose their own limits which
are more restricted.  But similarly to `string.new`, instantiating a
module with string literals may fail due to lack of memory resources,
even if the string size is formally within the limits.  However
`string.const` itself never traps when passed a valid literal offset.

### String positions

The optimal way to represent a position in a string is in terms of
code units in the encoding used internally by the WebAssembly run-time.
However we have to allow both for implementations that use WTF-8 and for
those that use WTF-16.  Also, some source languages will want to use
WTF-16 offsets.

As a compromise, we allow string positions to be expressed as `i32`
values, either in terms of WTF-8 code units or in WTF-16 code units.

WTF-8 and WTF-16 positions have different semantics:

 * WTF-8 positions address codepoints.  A string consisting of *N*
   codepoints will have *N*+1 distinct valid WTF-8 position values: one
   before each codepoint, and one at the end.  A position value of 0
   denotes the string start (before the first codepoint).  It follows
   that 0 may also denote the end of the string also, for zero-length
   strings.

 * WTF-16 positions address 16-bit code units.  A string consisting of
   *N* code units when encoded as WTF-16 will have *N*+1 distinct valid
   position: one before each 16-bit code unit, and one at the end.  As
   with WTF-8 positions, 0 denotes the string start.  There may be more
   WTF-16 positions in a string than there are WTF-8 positions, as
   there may be multiple WTF-16 code units per codepoint.
   A WTF-16 position is the same as the operand to JavaScript's
   *[`String.prototype.charCodeAt`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/charCodeAt).

A position is an offset into a string's contents.  Positions are
strictly ordered, and therefore can be compared against each other.

We expect WebAssembly implementations to represent strings using either
WTF-8 or WTF-16, and thus one of these encodings is "native" and the
other is "foreign".  Some implementations will want to use
[breadcrumbs](https://www.swift.org/blog/utf8-string/#breadcrumbs) to
project foreign positions to native positions.  A simple one-entry cache
may also suffice for some implementations.  Finally, we expect that many
source languages will process strings in chunks via in-memory encoding,
minimizing per-codepoint translation cost between foreign and native
offsets.

For WTF-16 positions, since position values are packed, we just provide
an accessor to get the last position in the string:

```
(string.end_wtf16 str:stringref)
  -> pos:i32
```
Return the position after the last code unit in the WTF-16 encoding of
the string *str*.

For WTF-8 positions, use the `string.advance_wtf8` and
`string.rewind_wtf8` seek instructions:

```
(string.advance_wtf8 str:stringref pos:i32 codepoints:i32)
  -> pos:i32, codepoints:i32
(string.rewind_wtf8 str:stringref pos:i32 codepoints:i32)
  -> pos:i32, codepoints:i32
```
Return a new WTF-8 position for the string *`str`* which is
*`codepoints`* codepoints after or before *`pos`*.  Also return the
number of codepoints by which the new position differs from the old,
which may be less than the *`codepoints`* parameter if end-of-string or
beginning-of-string would be reached.  The *`codepoints`* argument is
interpreted as a `u32`.

#### Invalid position values

All instructions which take string positions trap if the position is not
valid for the given string.

#### Position value limits

The maximum valid `i32` position value is 2<sup>31</sup>.

Evaluation of an instruction which would result in a WTF-16 position
value greater than 2<sup>31</sup> must trap.  In practice, no web
browser embedding allows for strings longer than 2<sup>31</sup>–1 WTF-16
code units, so no string from JavaScript is too large for the
instructions in this proposal, when processed as WTF-16.

Evaluation of an instruction which would result in a WTF-8 position
value greater than 2<sup>31</sup> must trap.  Note that depending on the
string contents, a position for a WTF-8 encoding may be numerically
lesser or greater than the corresponding WTF-16 position.

Future extensions of the `stringref` proposal along the lines of the
[memory64
proposal](https://github.com/WebAssembly/memory64/blob/main/proposals/memory64/Overview.md)
may allow for 64-bit variants of the position-using instructions, which
could relax these restrictions.

### Accessing string contents

All parameters and return values measuring a number of codepoints or a
number of code units represent these sizes as unsigned values.

```
(string.get_wtf8 str:stringref pos:i32)
  -> codepoint:i32
```
Return the codepoint at the WTF-8 *`pos`* in *`str`*, as an i32.  The
result will be a valid codepoint, and may be an isolated surrogate.  If
*`pos`* is at the end of *`str`*, the result is -1.

```
(string.get_wtf16 str:stringref pos:i32)
  -> codeunit:i32
```
Return the WTF-16 code unit at the WTF-16 *`pos`* in *`str`*, as an i32.
The result may be part of a surrogate pair, or may be an isolated
surrogate; the caller is responsible for decoding to codepoints as
needed.  If *`pos`* is at the end of *`str`*, the result is -1.

```
(string.measure_utf8 str:stringref pos:i32 codepoints:i32)
  -> bytes:i32
(string.measure_wtf8 str:stringref pos:i32 codepoints:i32)
  -> bytes:i32
```
Measure how many bytes would be necessary to encode the contents of the
string *`str`*, starting at position *`pos`*, to UTF-8 or WTF-8
respectively, limited to *`codepoints`*.  For `string.measure_utf8`, if
the *`codepoints`* following *`pos`* contain an isolated surrogate,
return -1.

The maximum number of bytes returned by `string.measure` is
2<sup>31</sup>-1.  If an encoding would require more bytes, it is as if
the contents can't be encoded at all (the return value is -1).

(Note that the limit on position values constrains strings to have a
maximum of 2<sup>31</sup>–1 codepoints, but it may be that such a string
could exceed even 2<sup>32</sup> bytes in a given encoding; e.g. a
string of 2<sup>31</sup>-1 `\uFFFF` codepoints would require
2<sup>32</sup>+2<sup>31</sup>-3 bytes when encoded as UTF-8.)

There is no `string.measure_wtf16`; `string.end_wtf16` provides the
total code unit length, and any position slice in that range is valid
and has a well-defined mapping to bytes.

```
(string.encode_utf8 str:stringref pos:i32 bytes:i32)
  -> bytes:i32
(string.encode_wtf8 str:stringref pos:i32 bytes:i32)
  -> bytes:i32
```

Encode the contents of the string *`str`* as UTF-8 or WTF-8,
respectively, starting at the WTF-8 byte offset *`pos`*, to memory at
*`ptr`*, limited to *`bytes`* bytes.  Return the number of bytes
written.  Note that no `NUL` terminator is ever written.  If any
codepoint cannot be encoded, trap.

The maximum number of bytes that can be encoded at once by
`string.encode` is 2<sup>31</sup>-1.  If an encoding would require more
bytes, it is as if the codepoints can't be encoded (a trap).

```
(string.encode_wtf16 str:stringref pos:i32 codeunits:i32)
```
Encode the *`codeunits`* code units of the string *`str`* as WTF-16,
respectively, starting at the WTF-16 code unit offset *`pos`*, to memory
at *`ptr`*.  *`pos`*+*`codeunits`* must be a valid position in *`str`*.

### Predicates

```
(string.eq_wtf8 a:stringref a-pos:i32 b:stringref b-pos:i32 codepoints:i32) -> i32
```
Return 1 if the strings *`a`* and *`b`* contain the same codepoint
sequence, starting at the WTF-8 positions *`a-pos`* and *`b-pos`*
respectively, limited to comparing up to *`codepoints`* codepoints.
Return 0 otherwise.

```
(string.eq_wtf16 a:stringref a-pos:i32 b:stringref b-pos:i32 codeunits:i32) -> i32
```
Return 1 if the strings *`a`* and *`b`* contain the same WTF-16 code
unit sequence, starting at positions *`a-pos`* and *`b-pos`*
respectively, limited to comparing up to *`codeunits`* code units.
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
  string.new_wtf8)
```

Generally speaking, this proposal only distinguishes between UTF-8 and
WTF-8 when encoding string contents to memory.  As this is a a decode
operation, the proposal just has a WTF-8 interface, as WTF-8 is a
superset of UTF-8.

### Make string from an array of WTF-8 code units in memory

```wasm
(func $string-from-wtf8n (param $ptr i32) (param $len i32) (result stringref)
  local.get $ptr
  local.get $len
  string.new_wtf8)
```

### Make string from UTF-16 in memory

```wasm
(func $string-from-utf16 (param $ptr i32) (param $units i32) (result stringref)
  local.get $ptr
  local.get $units
  string.new_wtf16)
```

This proposal doesn't distinguish between UTF-16 and WTF-16 at all;
rather it just deals in WTF-16, as most source languages that expose
16-bit code units to users actually expose WTF-16 strings.

### Number of codepoints in string

```wasm
(func $codepoint-length (param $str stringref) (result i32)
  local.get $str
  i32.const 0         ;; initial wtf8 offset: beginning of string
  i32.const -1        ;; advance by all codepoints
  string.advance_wtf8 ;; push new wtf8 offset, codepoints
  return)             ;; just return codepoints
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
  string.eq_wtf8)
```

Having to choose the encoding-specific variant of `string.eq` is a bit
funny in this case where we actually don't care about the encodings.

### Suffix, prefix comparisons

```wasm
(func starts-with-hey? (param $str stringref) (result i32)
  global.get $hey
  i32.const 0
  local.get $str
  i32.const 0
  i32.const 3
  string.eq_wtf8)

(func ends-with-howdy?/wtf8 (param $str stringref) (result i32)
  string.const "Howdy"
  i32.const 0
  local.get $str

  ;; Get wtf-8 offset 5 codepoints before end of $str:
  local.get $str
  ;; First get wtf-8 offset of end of $str...
  local.get $str
  i32.const 0
  i32.const -1
  string.advance_wtf8
  drop
  ;; ...then rewind by 5 codepoints.
  i32.const 5
  string.rewind_wtf8
  drop

  ;; Limit comparison to 5 codepoints.
  i32.const 5
  string.eq_wtf8)

(func ends-with-howdy?/wtf16 (param $str stringref) (result i32)
  (local $len i32)

  string.const "Howdy"
  i32.const 0
  local.get $str

  ;; Get position 5 code units before end of $str, knowing that "Howdy"
  ;; has 5 WTF-16 code units.
  local.get $str
  ;; First get cursor at end of $str...
  string.end_wtf16 $str
  ;; If position less than 5, it's not equal.
  local.tee $len
  i32.const 5
  (if l32.lt (then (i32.const 0) return))

  local.get $len
  i32.const 5
  i32.sub

  ;; Limit comparison to 5 code units.
  i32.const 5
  string.eq_wtf16)
```

Which version of `ends-with-howdy?` will a source language produce?
Neither is clearly better.  Probably source language that process
strings in terms of codepoints will produce the WTF-8 version whereas
those that process strings in terms of 16-bit code units will compile to
the WTF-16 version.

### Store a `stringref` without copying

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
  string.measure_utf8
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
  string.encode_wtf8               ;; push number of bytes written

  local.get $ptr
  i32.add
  i32.const 0
  i32.store8                       ;; write NUL

  local.get $ptr
  return)
```

Using `string.measure_utf8` ensures that the encoded string is a valid
unicode scalar value sequence.  How to handle invalid UTF-8 is up to the
user; instead of `unreachable` we could throw an exception.

If we meant to handle isolated surrogates, we could use
`string.measure_wtf8` instead.

### Stream over contents of string

Assume you have a 1024-byte array of memory at `$buf`.  This function
will encode isolated surrogates as WTF-8.

```wasm
(global $buf i32)
(func $process-wtf8 (param $ptr i32) (param $len i32))

(func $process-string (param $str stringref)
  (local $cursor i32)                ;; initial value of 0 is start
  (local $bytes i32)

  loop
    local.get $str
    local.get $cursor
    global.get $buf
    i32.const 1024
    string.encode_wtf8               ;; push bytes written
    local.tee $bytes
    (if i32.eqz (then return))       ;; if no bytes encoded, done
    local.get $bytes
    local.get $cursor
    i32.add
    local.set $cursor

    global.get $buf
    local.get $bytes
    call $process-utf8
  end)
```

### Stream over UTF-16 code units of string, handling isolated surrogates

This function is probably slower than encoding chunks of the string to
WTF-16 in linear memory, for longer strings.

```wasm
(func $have-code-unit (param $codeunit i32))

(func $process-string (param $str stringref)
  (local $cur i32)
  (local $len i32)

  local.get $str
  string.end_wtf16
  local.set $len

  block $done
    loop $loop
      local.get $cur
      local.get $len
      i32.ge
      br_if $done

      local.get $str
      local.get $cur
      string.get_wtf16
      call $have-code-unit

      i32.const 1
      local.get $cur
      i32.add
      local.set $cur
    end
  end)
```

### Stream over codepoints of string, handling isolated surrogates

This function is probably slower than encoding chunks of the string to
WTF-8 in memory, for longer strings.

```wasm
(func $have-codepoint (param $codepoint i32))

(func $process-string (param $str stringref)
  (local $cur i32)
  (local $ch i32)

  block $done
    loop $loop
      local.get $str
      local.get $cur
      string.get_wtf8
      local.tee $ch

      i32.const -1
      i32.eq
      br_if $done

      local.get $ch
      call $have-codepoint

      local.get $str
      local.get $cur
      i32.const 1
      string.advance_wtf8
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

### What is the expected implementation on non-browser run-times?

Assuming that the non-browser implementation uses WTF-8 as the native
string representation, then a `stringref` is a pointer, a length, and a
reference count.  Some implementations may also want to keep a flag
indicating whether a string is valid UTF-8.  Most implementations will
also want to implement a map from UTF-16 position to UTF-8 position via
[breadcrumbs](https://www.swift.org/blog/utf8-string/#breadcrumbs).
UTF-8 position validation is ensuring the cursor is less than or equal
to the string byte length, and that `(ptr[cursor] & 0xb0) != 0x80`.
Encoding UTF-8 is just `memcpy`.

### What's the expected implementation in web browsers?

We expect that web browsers use JS strings as `stringref`.  WTF-16
access uses the same primitives exposed to JavaScript.  Iterating the
codepoints of a string using the WTF-8 interfaces might use a small
per-instance cache that associates the most recent WTF-16 and WTF-8
offsets for the last-accessed string or strings.

There is a possibility that some web browsers may eventually switch from
the one-byte/two-byte representation to WTF-8 with breadcrumbs, which
would make browsers follow the same strategy as the non-browser case.

Without a cache, accessing each codepoint in a string via
`string.get_wtf8` would be quadratic.  Perhaps this proposal should
specify time complexity bounds for some kinds of string algorithm, for
example visiting each codepoint in a string.

### What are the expected performance characteristics of the WTF-8 and WTF-16 interfaces?

We expect that compilers that emit WTF-8 instructions prefer chunks of
`string.encode_wtf8` over `string.get_wtf8` / `string.advance_wtf8`.
`string.get_wtf8` / `string.advance_wtf8` will be faster on WebAssembly
implementations that use WTF-8 internally.

We expect that compilers that emit the WTF-16 interface place more
importance on `string.get_wtf16`.  Implementations should ensure that
`string.get_wtf16` runs in near-linear time, even on systems that
represent strings internally as WTF-8.

### Could abstract the concept of a string position?

The question is, if we see strings as sequences of codepoints that can
be seeked around in, what if we defined an abstract time for a cursor
into a string?  Such a cursor could hold onto the string and so avoid
any need for position validation, and could abstract over the
differences between implementations that use WTF-8 or WTF-16
internally.

One consideration is that whatever we do, some source languages will
need WTF-16 codepoint access (`string.get_wtf16`).  This makes abstract
cursors less attractive because they are not comprehensive.  Abstract
cursors could replace uses of WTF-8 string positions which are really
about accessing the codepoints of a string and only incidentally about
UTF-8.

Defining a string cursor type is tricky though -- would you allow them
to be stored to globals?  Passed as parameters?  To JavaScript?  How
would you reference them?  Would they be heap objects with identity, or
reference types which somehow have no identity?  Could you store them to
memory?  (Probably not.)  Just using `i32` WTF-8 and WTF-16 offsets does
solve these issues.

This is probably the biggest open question of this proposal.  See
https://github.com/wingo/wasm-strings/issues/6,
https://github.com/wingo/wasm-strings/issues/11,
https://github.com/wingo/wasm-strings/issues/21, and
https://github.com/wingo/wasm-strings/issues/24.

### Is the `stringref` type nullable?

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
JavaScript is bug-prone.  Using `stringref` would eliminate questions of
memory ownership, reducing the risk of use-after-free, data corruption,
write overruns, and privileged data leakage.

### Doesn't the `encode` interface imply some copying overhead?

Yes, but this overhead is offset by the inlined code unit processing on
the WebAssembly side, and somewhat hidden by CPU caches.  (At some point
we will need to measure this.)

It's possible to access individual codepoints in a string using the
WTF-8 interface.  This will be faster on some implementations than
others.  It's also possible to access individual WTF-16 code units,
which should be fast on all implementations.

Note however that individual codepoint access isn't quite "raw";
JavaScript strings, for example, are quite polymorphic, and even in
non-browser WTF-8 implementations there will still be ropes and slices.

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
although you could implement structs and arrays via `externref` and
imports, if you did that you might as well compile to JavaScript instead
of WebAssembly.  It should be possible to make a WebAssembly program
that uses reference-typed strings (because almost all such programs
would have strings) without relying on any JavaScript at all.

Also, the evolutionary endpoint of an `externref`-and-imports strategy
is a JavaScript-specific string interface.  Without any broader
WebAssembly platform concern, strings-using WebAssembly code would find
itself relying on details of JavaScript's string representation, for
example having the only interface be to process strings one code unit at
a time instead of one codepoint at a time.  This is not a good platform
outcome.

Finally, though the WebAssembly platform should be able to stand alone,
it should also interoperate smoothly with hosts, especially JavaScript
on the web.  This rules out any implementation of strings in terms of
reference-typed structs and arrays: not only would such an
implementation be likely slower than the host's strings, it would also
be incompatible.  On the web, WebAssembly and JavaScript should use the
same string implementation.

On the performance side, we expect that `stringref` will be
faster than `externref`+imports:

 1. Whereas an `externref` might need to be a tagged union, a
    `stringref` can be an unpacked pointer.
 2. WebAssembly instructions are likely faster and less of an
    optimization barrier than callouts to imports.
 3. Run-time helper code for WebAssembly instructions is probably
    implemented in C++/Rust/etc more directly, resulting in more
    predictable performance than e.g. an encoder implemented in JS (for
    web embeddings).
 4. Reading string contents via
    `string.encode_wtf8`-then-process-inline is likely faster than
    calling out to JavaScript to read code units one at a time.
    WebAssembly-to-JavaScript calls are cheap but not free.

On the other hand, it's true that JS run-time routines can use adaptive
JIT techniques to possibly inline representation-specific accessors.
This is of limited use though for run-time routines with many different
call sites.

On the reliability and security side, adding `stringref` to WebAssembly
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
anything, even GC-managed data, zero-copy communication of
reference-typed strings between components is strictly out of scope
(though this operation may be zero-copy in practice; see below).

From the perspective of the component model, reference-typed strings are
rather an *intra-component* concern.  A component may be composed
internally of a number of WebAssembly modules, as well as possible host
facilities such as JavaScript.  The zero-copy properties provided by
`stringref` are only assured on to the inter-module, intra-component
boundaries of a program.

That said, strings in the abstract are an important data type, and
relate to interface types (a WebAssembly proposal based on the component
model).  Obviously you will want to be able to use `stringref` with
interface types.  The shared-nothing design choice of the component
model then implies that `stringref` contents should be copied when they
cross a component boundary.

Incidentally, for inter-component interfaces that deal in strings, the
component model specifies that abstractly, [strings are sequences of
unicode scalar
values](https://github.com/WebAssembly/interface-types/issues/135).
This implies that some JavaScript strings can't traverse a component
boundary, because of the potential for isolated surrogates, and also
implies an eager check that a `stringref` is a valid USV sequence, for
an interface-typed call.  In practice this is not a problem because the
`stringref` contents are being copied anyway and so can be validated at
the same time.

Interface types are used to specify a WebAssembly function's signature
in an abstract way.  This signature should then be compiled down to a
concrete adapter function specialized to the data representations used
by the caller and the callee.  The instruction set in this proposal can
be used to implement the adapter function for passing a `stringref` as a
string; assuming that the adapter function is generated in such a way
that it has access to the target memory, `string.encode` can implement
the copy and validation at the same time.  `string.new` would be the
implementation of getting a `stringref` from an interface-typed string
value.

Of course, because a `stringref` is immutable, whether it is copied or
not on a component boundary or during a call to an interface-typed
function is an implementation detail.  Some implementations of the
component model may wish to copy in all cases, for memory usage
accounting reasons.  Others will apply a zero-copy strategy when
possible, for example when both the caller and the callee of an
interface are implemented with `stringref`.  In the zero-copy case,
however, hosts have to eagerly verify that the string is a valid USV
sequence; see
[isUSVString](https://github.com/guybedford/proposal-is-usv-string).
