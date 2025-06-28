# OSN Specification

[中文](README_zh.md) | English

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> OSN (Object Serialization Notation) is a text-based format for object serialization, designed to provide a data representation that is easy to read, parse, and generate.  
> Its design is inspired by JSON but improved for better readability and flexibility.

OSN uses a key-value pair structure (`key:value`) to represent members. The key is on the left side of the colon (`:`), and the value is on the right.

- A key can be any single-line text. If the key's text consists only of letters, numbers, underscores, and hyphens, the surrounding double quotes (`"`) can be omitted. Otherwise, the key must be enclosed in double quotes and treated according to string rules (see the "Strings - Single-Line Strings" section).
- A value can be a number, boolean, string, array, object, or `null`.

Any number of key-value pairs can be written per line. If a line contains multiple key-value pairs, they must be separated by a comma (`,`). If a line contains only one key-value pair, the trailing comma can be omitted. A trailing comma is permitted after the last key-value pair. Excess whitespace between key-value pairs is ignored.

For a complete example, see: [sample.osn](./sample.osn)

## Comments

OSN supports single-line comments. A comment starts with `//` and extends to the end of the line. The content of a comment is ignored by the parser. Comments can appear anywhere except within a multi-line string.

Example:

```js
// This is a single-line comment, which can occupy its own line.
Key: "Value" // Comments can also follow a key-value pair.
```

## Data Types

### Numbers

OSN supports integers and floating-point numbers. It allows for scientific notation as well as binary, octal, and hexadecimal representations. Underscores (`_`) can be used as separators within numbers to improve readability.

When using scientific, binary, octal, or hexadecimal notation, the letter part is case-insensitive.

Example:

```js
IntegerValue: 42
FloatValue: 3.14
NumberFieldScientific: 3.14E-10
BinaryValue: 0b0010_1010
OctalValue: 0o52
HexValue: 0x2A
```

### Booleans

OSN uses **lowercase** `true` and `false` as boolean literals.

Example:

```js
BooleanValue1: true
BooleanValue2: false
```

### Strings

OSN supports single-line and multi-line strings. Embedding environment variables is not supported within strings (see the "Environment Variables" section).

**Single-line strings** are enclosed in double quotes (`"`) and support escape characters, following the same rules as JSON (as per [RFC 8259]).

**Multi-line strings** are enclosed in triple quotes (`"""`). All characters within them are treated as literals; escape sequences are not supported. The content of a multi-line string cannot start on the same line as the opening triple quote, and each line of text must begin with a pipe character (`|`). All whitespace before the pipe character on each line is ignored. All content after the pipe character, including other pipe characters and leading/trailing whitespace, is preserved. A line containing only a pipe character (with optional leading/trailing whitespace) represents an empty line.

Example:

```js
SingleLineStringField: "Hello World!",
SingleLineStringFieldWithEscape: "says:\n\"Hello!\"",
MultiLineStringField: """
                      |This is a multi-line string that can span multiple lines.
                      |All characters within this """ block are treated as literals;
                      |no escapes or comments are processed.
                      |Each line must start with a pipe character `|` to control indentation.
                      |// This is part of the string, not an OSN comment.
                      |The next line is an empty line:
                      |
                      """,
```

### Arrays

OSN uses square brackets (`[]`) to represent arrays. Array elements are placed within the brackets and can be of any value type. If there are multiple elements on a single line, they must be separated by a comma (`,`). If there is only one element on a line, the comma can be omitted. Excess whitespace between elements is ignored.

Example:

```js
ArrayOfIntegers: [1, 2, 3]
ArrayOfStrings: [
    "你好"
    "Hello"
]
ArrayOfArrays: [
    [1, 2]
    [3, 4]
]
```

### Objects

OSN uses curly braces (`{}`) to represent objects. Object members, expressed as key-value pairs, are placed within the braces. In programming languages, an OSN object can be parsed into the language's native object or dictionary equivalent.

Keys at the same level within an object must be unique, but a child key (a key in a nested object) can be the same as a parent key.

An OSN document itself represents an object; the curly braces for the top-level object can be omitted.

Example 1:

```js
ObjectField: {
    Field1: "Value",
    Field2: 42,
    Field3: [1, 2, 3],
    Field4: {
        SubField1: "SubValue1",
        SubField2: true
    }
    "Special Key": "Keys with special characters must be wrapped in double quotes."
}
```

**Member Accessor**: The dot (`.`) character can be used as a member accessor to access object members. Whitespace before and after the member accessor is ignored. A dot character within double quotes is not treated as a member accessor.

Example 2 (equivalent to Example 1 in this section):

```js
ObjectField.Field1: "Value"
ObjectField.Field4.SubField1: "SubValue1"
ObjectField: {
    Field2: 42,
    Field3: [1, 2, 3],
    Field4: {
        SubField2: true
    }
}
```

### null

OSN uses the lowercase literal `null` to represent a null object or no value. When `null` is used for a non-nullable type, the parser should, based on its configuration, either resolve it to a default value or report an error.

Example:

```js
ObjectField: null
IntegerField: null // May be parsed as 0 or cause an error, depending on parser settings.
```

## Directives (TBD)

OSN supports directives to specify parser behavior or provide metadata. Directives are optional, but the same directive cannot be applied to the same member more than once. Directives are not strictly enforced; parsers should allow configuration to ignore, issue a warning, or report an error when OSN content does not comply with a directive.

A directive starts with an `@` symbol, followed by the directive name and its arguments.

  - The directive name is always in **lowercase**.
  - Directive arguments must be placed within parentheses (`()`), are case-sensitive, and should **not** be enclosed in double quotes.
  - For directives without arguments, the parentheses after the name can be omitted.

Directives are categorized as Document Directives, Tagging Directives, Reference Directives, and Environment Variables.

### Document Directives

Document directives are placed before all key-value pairs in an OSN document and apply to the entire document.

#### @omd()

The `@omd()` directive specifies the OMD (Object Model Definition) file for the OSN document. The argument is the path to the OMD file. This path can include environment variables (see "Environment Variables") and can be a relative, absolute, or remote path. The parser should provide options to control whether remote paths are allowed and to handle related security issues.

The OMD file defines object types, field names, field types, field constraints, etc. Definitions in the OMD file may conflict with inline directives in the OSN document. In case of a conflict, inline directives take precedence. The parser should allow configuration to issue a warning or an error for such conflicts.

If an OSN document does not specify an `@omd()` directive, but an OMD file named `.omd` exists in the same directory as the OSN file, that file will be used as the default OMD.

Example:

```js
// ${OMD_ROOT} is an environment variable that will be replaced with its actual value at parse time.
@omd(${OMD_ROOT}/sample.omd)
// Key-value pairs...
```

### Tagging Directives

Tagging directives mark specific key-value pairs in an OSN document. They are placed on the line immediately preceding the tagged key-value pair or at the beginning of the same line. Any amount of whitespace is allowed between a tagging directive and its target key-value pair.

#### @type()

The `@type()` directive specifies the runtime type of the value in the tagged key-value pair. The argument is the fully qualified name of the type.

Example:

```js
@type(MyNamespace.MyType)
ObjectField: {
    // At runtime, ObjectField will be parsed as an object of type MyNamespace.MyType.
    // ...
}
```

#### @notnull

The `@notnull` directive marks the value of a key-value pair as non-nullable. It takes no arguments, and the parentheses can be omitted.

By default, values in OSN can be `null`. When a key-value pair is marked with `@notnull`, if its value is `null`, the parser should allow configuration to determine whether to report an error.

Example:

```js
// The parser may report an error for the following, depending on its settings.
@notnull() StringField: null
@notnull // Parentheses are optional.
ObjectField: null
```

### Reference Directives

Reference directives are used to reference the value of a member from the current or another OSN document. They are placed where a value would normally appear.

#### @ref()

The `@ref()` directive references the value of a member from the current or another OSN document. The argument is the path to the target member's key, in the format `OsnDocumentPath::KeyPath`, where:

  - `OsnDocumentPath` is the path to the target OSN document. It can contain environment variables (see "Environment Variables") and can be a relative, absolute, or remote path.
      - If this part is empty, it refers to the current document.
      - The parser should provide options to control whether remote paths are allowed and to handle related security issues.
  - `KeyPath` is the path to the target member's key within its OSN document, using the member accessor (`.`) to separate keys at different levels (see "Objects - Member Accessor").
      - If a key name contains special characters, it must be enclosed in double quotes (see "Key" rules).
      - To allow for `null` values in the reference chain, the safe member accessor (`?.`) can be used instead of the standard member accessor (`.`). This means if the member is `null`, the entire reference resolves to `null`.
      - When the safe member accessor is not used, the parser should report an error for any `null` value encountered in the reference chain.
  - `::` is the separator between `OsnDocumentPath` and `KeyPath`.

The parser should be able to handle circular references.

Example:

```js
// References the value of ObjectField.Field1 from other.osn. The parser will report an error if ObjectField is null.
Key1: @ref(${COMMON_OSN_ROOT}/other.osn::ObjectField.Field1)
// Uses the safe member accessor. If ObjectField is null, Key2's value will be null, and the parser will not report an error.
Key2: @ref(${COMMON_OSN_ROOT}/other.osn::ObjectField?.Field1)
```

### Environment Variables

OSN supports referencing environment variables within the value of a key-value pair using the `${}` syntax. The name of the environment variable should be placed inside the curly braces. The name can only consist of numbers, letters, and underscores; do not enclose the environment variable name in double quotes.

During parsing, the parser will replace the environment variable reference with its actual runtime value. The parser should allow configuration to handle undefined environment variables by either resolving them to a default value or reporting an error. The parser must handle security issues related to environment variables.

Example:

```js
EnvironmentVariable: ${GAME_VERSION}
```