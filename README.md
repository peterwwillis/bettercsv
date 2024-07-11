# bettercsv

CSV files aren't a great format for storing data. They are simple to think about, but difficult to use in many cases, and lack of standardization makes it worse.

This data format is an attempt to tackle some common problems with CSV files, while retaining backwards compatibility with CSV, and extending it to be more useful in the 21st century.

Here's a better version of CSV.

---

## Design specs

- **Author: Peter Willis**
- **Date: 7/11/2024**
- **Better CSV Specification Version: 1**

## Design goals

1. Backwards compatible with CSV
2. Easy to think about (K.I.S.S.)
3. Difficult enough to parse that programmers will have to refer to the spec (rather than just looking at a raw file) in order to implement a parser
4. Future-proof
5. Add features people might want in a modern tabular data format

### Backwards Compatibility with CSV format
The format's semantics are backwards-compatible with RFC4180.

The format uses special identifiers for rows specific to this format, and special columns to identify rows used for metadata.

Therefore you can mix this format into an existing CSV file and it should still be parseable by both a modern CSV file parser, and a *Better CSV* parser.

#### Delimiter
All columns will be separated with a ",", <TAB>, or other CSV-compatible delimiter.

#### Rows
Every header and data row will have a first column entry starting with "!BCv", followed by a number, which is the version of the *Better CSV* specification.

At time of writing, the version of this specification is **1**.

#### Fields
Every field will be double-quoted, regardless of whether they contain any data.

Conforming with RFC4180, if double-quote characters exist within a field, they must be escaped by preceeding them with another double-quote (e.g. `""`).

##### Metadata Rows
If a row's first column entry starts with "#!BCv" (followed by the specification version), that row encodes metadata about a column or row (or both).

The second column must be a column identifier, row identifier, or a combination of both separated by a "#" character.

Rows begin at 0, which would be the header row (if one exists). However, there should not be a use for applying metadata to the header row, so typically this digit should be `1` or greater (assuming there is a header row; if there is no header row, this should be `0`).

Columns begin at 1, because the first column is always just the 

###### Column Identifier
The column is the primary metadata identifier, as the main purpose of metadata is to define rules for entire columns.

Column identifiers must be a digit (`12`), range of digits (`2-4`), or set of digits (`3,4,7`) identifying column(s).

If the column identifier is `*`, then the metadata applies to all columns.

If you want metadata to apply to all columns except for one column, prefix the column identifier with a `!` character (ex. `!12` will apply to all columns except column 12).

###### Row Identifier
The row identifier exists to specify metadata that applies to specific rows+columns.

Row identifiers must be a digit or range of digits followed by a "#" character followed by a digit (`12#7`) or range of digits (`12#7-9`), identifying both row(s) and column.

Row identifiers can exist within the individual entries in a column set (ex. `1,2-3,4#7,8-9#11-13`).

If you want metadata to apply to all rows, simply leave out the row identifier (ex. `12` will apply to all rows for column 12).

If you want metadata to apply to all rows except one row, prepend the row identifier with a `!` character (ex. `12#!7` will apply to column 12 for all rows except row 7)

### Field Types
The type of a field is identified by an index from 0-255.

Types can be combined to allow parsing to match more than one type, by following one type index with "+" and another type index.

Types can declare invalid type parsing by following one type with the "-" character and another type index. (Ex: for type `1+4-2`, the field data will be considered invalid if it contains numbers)

| Type Index | Type Descriptor | Example |
| ---        | ---             | ---     |
| 0  | Null type. Column is neither empty nor full, but is disabled. | Null |
| 1  | Alphabet type. Characters of the alphabet only. Case-sensitive. Locale/encoding specific. | `ABCDEFGHIJKLMNOPQRSTUVWXYZ` |
| 2  | Numeric type, decimal. Locale/encoding specific. | `0123456789` |
| x  | Boolean type. Only accepts values "0" or "1". | `0`, `1` |
| 3  | Symbol type. Printable special characters and symbols. Locale/encoding specific. | `~!@#$%^&*()_+`-={}|[]\:";'<>?,./` |
| 4  | Email type. MUST parse to ONLY RFC6532 semantics. (Hey, you, code monkey! Don't be lazy, follow the RFC!) | |
| 5  | Phone type. Characters supported include digits 0-9, letters A-D, and special characters "+", "-", ",", "\*", and "#". The "+" character may ONLY come as the first entry in the field, and must be followed by a valid global country code. Character "-" is supported but stripped during parsing as it is only a visual identifier. Case-insensitive. Locale/encoding specific. | +1-555-555-5555,,3,654654654 |
| 6  | JSON string type. A string encoded for a JSON string type element, not a whole JSON document. | "Hello\nWorld\t\ud83d\ude1b" |
| 7  | Base32 type. MUST parse to ONLY RFC4648 Section 6 semantics. Useful for fields that require a restricted character set. | `JBSWY3DPEBLW64TMMQFA====` |
| 8  | Base64 type. MUST parse to ONLY RFC4648 Section 4 semantics. Even though this encoding incurs a 33% size increase, testing shows some languages have an extremely fast implementation of this encoding, so it may be preferred at the expense of additional size. | `SGVsbG8gV29ybGQK` |
| 9  | BCenc85 type. This is a proprietary encoding similar to Adobe's Ascii85 format (ideal for binary files), with the exception that the character set used is ASCII characters 35-120. This preserves the advantages of the original format, but skips the `!` and `"` characters, which allows it to be enclosed between double-quote characters without additional escaping. This is more space-efficient than Base64 and retains backwards compatibility with CSV files, but is less space-efficient than the BCencraw data type. | 
| 10 | BCencraw data type. Any character at all is allowed, except for `"`, which must be escaped with an additional double-quote (ex. `""`). Use of this data type will lead to some extremely ugly-looking files, and definitely **will not** be compatible with any existing CSV parser. However, for extremely large fields, this will result in much less wasted file space, less memory use, and should be much faster to parse (assuming a fast parser e.g. a C library). If you use this data type, rename your file extension to `.bcsv` to prevent confusion. | |

### Metadata

### Sample 1

```csv
"!BCv1", "FullName", "FirstName", "LastName", "Address1", "Address2", "Address3", "Address4", "Address5", "PhoneNumber1", "PhoneNumber2", "Email", "Username", "Password"
"#!BCv1", "0", "Type", "1", "Encoding", "UTF-8", "Min", "0", "Max", "50"
"#!BCv1", "1-2", "Type", "1", "Encoding", "UTF-8", "Min", "0", "Max", "25"
"#!BCv1", "3-7", "Type", "1+2+3", "Encoding", "UTF-8", "Min", "0", "Max", "25"
"#!BCv1", "8-9", "Type", "2", "Encoding", "UTF-8", "Min", "0", "Max", "25"
"#!BCv1", "10", "Type", "01", "Encoding", "UTF-8", "Min", "0", "Max", "25"
"#!BCv1", "11", "Type", "1", "Encoding", "UTF-8", "Min", "0", "Max", "25"
"#!BCv1", "12", "Type", "7", "Encoding", "UTF-8", "Min", "8"
"!BCv1", 
```
