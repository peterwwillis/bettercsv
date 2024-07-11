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

1. Easy to think about (K.I.S.S.)
2. Difficult enough to parse that programmers will have to refer to the spec (rather than just looking at a raw file) in order to implement a parser
3. Backwards compatible with CSV (RFC4180), so *Better CSV* can be added onto existing CSV parsers as an extension.
4. Add features people might want in a modern tabular data format
5. Future-proof

### Backwards Compatibility with CSV format
The format's semantics are backwards-compatible with RFC4180.

The format uses special identifiers for rows specific to this format, and special columns to identify rows used for metadata.

Therefore you can mix this format into an existing CSV file and it should still be parseable by both a modern CSV file parser, and a *Better CSV* parser.

### New functionality
Every row that includes *Better CSV* functionality is identified by one of two ways:

- **Column-based identification**
  - If there exists a header row, and it includes a column named `!BCv`, this is called the *Better CSV version column*, and that column will be inspected for each row.
    - If the row's *Better CSV version column* field is a positive number, this is taken as a *Better CSV* version number.
    - If the row's *Better CSV version column* field is `#` followed by a positive number, this is taken as a metadata row, and a *Better CSV* version number.
  - If no such header column exists, all records are assumed to be just regular CSV records.

- **Row-based identification**
  - If the first column field contains the string `!BCv` followed by a number, this row is considered to have *Better CSV* functionality with a specific version number.
  - If the first column field contains the string `#!BCv` followed by a number, this row is considered to be a *Better CSV* metadata record with a specific version number.
  - Otherwise the row is considered just a regular CSV row.

Using this method of identification, rows can be identified as using specific versions of the spec. The same file can grow and change over time, incorporating different versions of the spec for different rows, while maintaining backwards compatibility.

#### Fields
Every field MUST be double-quoted, regardless of whether it contains any data.

Conforming with RFC4180, if double-quote characters exist within a field, they must be escaped by preceeding them with another double-quote (e.g. `""`).

#### Metadata Rows
Some rows only encode metadata about other rows/columns. These are metadata rows.

To determine which rows contain metadata, check the **New functionality** section above.
 - If **Column-based identification** was used, each column in a row is considered a data column, except for the *Better CSV version column*.
 - If **Row-based identification** was used, the second column is the first data column.

The first data column is always a column identifier, row identifier, or a combination of both separated by a `#` character.

Row and column identifiers are the "real" row or column, as seen by any CSV parser.
 - If there's a header column, your row identifier would start at `1` as that's the first data row.
 - If you have no header column, your row identifier starts at `0`.
 - If you used **Column-based identification**, and there's a header row, then one of your columns is not going to be a data column (as it has the *Better CSV* version identifier), so your first column could be 0, assuming you didn't make the first column the *Better CSV* version identifier column.
 - If you used **Row-based identification**, then your first column is always the `!BCv` identifier, so your data columns will start at `1`.

In a metadata row, all data columns after the first data column are effectively "key/value" pairs, where one column is the key, and the next column is the value, and so on. The *Better CSV version column* is always skipped (if it exists). With this design, future versions of the spec can include new metadata key/value pairs, and each metadata row can identify what version of the spec it applies to.

##### Column Identifier
The column is the primary metadata identifier, as the main purpose of metadata is to define rules for entire columns.

Column identifiers must be a digit (`12`), range of digits (`2-4`), or set of digits (`3,4,7`) identifying column(s).

If the column identifier is `*`, then the metadata applies to all columns.

If you want metadata to apply to all columns except for one column, prefix the column identifier with a `!` character (ex. `!12` will apply to all columns except column 12).

##### Row Identifier
The row identifier exists to specify metadata that applies to specific rows+columns.

Row identifiers must be a digit or range of digits followed by a `#` character followed by a digit (`12#7`) or range of digits (`12#7-9`), identifying both row(s) and column.

Row identifiers can exist within the individual entries in a column set (ex. `1,2-3,4#7,8-9#11-13`).

If you want metadata to apply to all rows, simply leave out the row identifier (ex. `12` will apply to all rows for column 12).

If you want metadata to apply to all rows except one row, prepend the row identifier with a `!` character (ex. `12#!7` will apply to column 12 for all rows except row 7)

##### Field Types
A metadata row column with the string "Type" is followed by a column string with the field type to use.

The type of a field is identified by an index number from 0-65535.

Types can be combined to allow parsing to match more than one type, by following one type index with `+` and another type index. They are parsed serially.
 - Within a parsing application, parsing a data type results in a particular type of object.
 - That new object is then passed onto the next type specified.
 - This allows you to parse a field via one type into one object, and then pass that new object onto another type parser, resulting in yet another object.
 - So for example, by using type `42+50+9`, a field will be parsed as Base64 data into a binary object, and then parsed as GZip into yet another binary object, and finally parsed into a string-type object (in the final case, using the "Encoding" metadata feature).
 - Creative use of type specification allows you to programmatically encode and decode data on the fly through various data formats/types.

Types can declare invalid type parsing by following one type with the `-` character and another type index. (ex: for type `1+31-3`, the field data will be considered invalid if it contains numbers)

| Type Index | Type Descriptor | Example |
| ---        | ---             | ---     |
| 0  | Null type. Column is neither empty nor full, but is disabled. | Null |
| 1  | Alphabet type. Characters of the alphabet only. Case-sensitive. Locale/encoding specific. | `ABCDEFGHIJKLMNOPQRSTUVWXYZ` |
| 2  | Symbol type. Printable special characters and symbols. Locale/encoding specific. | `~!@#$%^&*()_+\`-={}|[]\:";'<>?,./` |
| 3  | Numeric type, integer. Locale/encoding specific. | `0123456789` |
| 4  | Numeric type, float. Locale/encoding specific. | `0.133235242` |
| 5  | Boolean type. Only accepts values `0` or `1`. | `0`, `1` |
| 9  | String type. Accepts any string. Case-insentitive. Locale/encoding specific. | `Honey, I shrunk the kids! 😱` |
| 10 | Date type. | |
| 11 | DateTime type. | |
| 12 | Duration type. | |
| 13 | Day of month type. | |
| 14 | Day of week type. | |
| 15 | Day of year type. | |
| 20 | Hour type. | |
| 21 | Minute type. | |
| 22 | Second type. | |
| 23 | Millisecond type. | |
| 24 | Nanosecond type. | |
| 25 | AM or PM type. | |
| 26 | Timezone type. | |
| 27 | Seconds since epoch type. | |
| 30 | URI type. | |
| 31 | Email type. MUST parse to ONLY RFC6532 semantics. (Hey, you, code monkey! Don't be lazy, follow the RFC!) | |
| 32 | Phone type. Characters supported include digits 0-9, letters A-D, and special characters `+`, `-`, `,`, `\*`, and `#`. The `+` character may ONLY come as the first entry in the field, and must be followed by a valid global country code. Character `-` is supported but stripped during parsing as it is only a visual identifier. Case-insensitive. Locale/encoding specific. | `+1-555-555-5555,,3,654654654` |
| 40 | JSON string type. A string encoded for a JSON string type element, not a whole JSON document. | `Hello\nWorld\t\ud83d\ude1b` |
| 41 | Base32 type. MUST parse to ONLY RFC4648 Section 6 semantics. Useful for fields that require a restricted character set. | `JBSWY3DPEBLW64TMMQFA====` |
| 42 | Base64 type. MUST parse to ONLY RFC4648 Section 4 semantics. Even though this encoding incurs a 33% size increase, testing shows some languages have an extremely fast implementation of this encoding, so it may be preferred at the expense of additional size. | `SGVsbG8gV29ybGQK` |
| 43  | BCenc85 type. This is a proprietary encoding similar to Adobe's Ascii85 format (ideal for binary files), with the exception that the character set used is ASCII characters 35-120. This preserves the advantages of the original format, but skips the `!` and `"` characters, which allows it to be enclosed between double-quote characters without additional escaping. This is more space-efficient than Base64 and retains backwards compatibility with CSV files, but is less space-efficient than the BCencraw data type. | 
| 44 | BCencraw data type. Any character at all is allowed, except for `"`, which must be escaped with an additional double-quote (ex. `""`). Use of this data type will lead to some extremely ugly-looking files, and definitely **will not** be compatible with any existing CSV parser. However, for extremely large fields, this will result in much less wasted file space, less memory use, and should be much faster to parse (assuming a fast parser e.g. a C library). If you use this data type, rename your file extension to `.bcsv` to prevent confusion. | |
| 50 | GZip data type. Fields MUST NOT use this type alone; use after a different field type that encodes binary data. | |
| 60 | JPEG data type. Fields MUST NOT use this type alone; use after a different field type that encodes binary data. | |
| 61 | GIF data type. Fields MUST NOT use this type alone; use after a different field type that encodes binary data. | |
| 62 | PNG data type. Fields MUST NOT use this type alone; use after a different field type that encodes binary data. | |

##### Character Encoding
A metadata row column with the string "Encoding" is followed by a column string with the encoding to use.

This field only applies to fields whose type allow a character encoding, such as strings.

Encoding string value is whatever encoding you want the field to have (ex. "UTF-8", "iso-8859-1", etc).

##### Minimum Length
A metadata row column with the string "Min" is followed by a column number with the minimum length of field data.

##### Maximum Length
A metadata row column with the string "Max" is followed by a column number with the maximum length of field data.

### Sample 1: Row-based identification

```csv
"!BCv1","FullName","FirstName","LastName","Address1","Address2","Address3","Address4","Address5","Address6","PhoneNumber1","PhoneNumber2","Email","Username","Password","Photo"
"#!BCv1","1","Type","1","Encoding","UTF-8","Min","0","Max","50"
"#!BCv1","2-3","Type","1","Encoding","UTF-8","Min","0","Max","25"
"#!BCv1","4-8","Type","1+2+3","Encoding","UTF-8","Min","0","Max","25"
"#!BCv1","9-10","Type","32","Encoding","UTF-8","Min","0","Max","25"
"#!BCv1","11","Type","31","Encoding","UTF-8","Min","0","Max","25"
"#!BCv1","12","Type","1+2+3","Encoding","UTF-8","Min","0","Max","25"
"#!BCv1","13","Type","1+2+3","Encoding","UTF-8","Min","8"
"#!BCv1","14","Type","42+62"
"!BCv1","Peter Willis","Peter","Willis","1234 Full Rd","","Somecity","Somestate","12345","USA","1","+1-555-555-5555","","foo@foo.foo","someusername1","h873hf8w7f82g8rfd2wf",""
```

### Sample 2: Column-based identification

```csv
"FullName","FirstName","LastName","Address1","Address2","Address3","Address4","Address5","Address6","!BCv","PhoneNumber1","PhoneNumber2","Email","Username","Password","Photo"
"1","Type","1","Encoding","UTF-8","Min","0","Max","50","#1"
"2-3","Type","1","Encoding","UTF-8","Min","0","Max","25","#1"
"4-8","Type","1+2+3","Encoding","UTF-8","Min","0","Max","25","#1"
"9-10","Type","32","Encoding","UTF-8","Min","0","Max","25","#1"
"11","Type","31","Encoding","UTF-8","Min","0","Max","25","#1"
"12","Type","1+2+3","Encoding","UTF-8","Min","0","Max","25","#1"
"13","Type","1+2+3","Encoding","UTF-8","Min","8","","","#1"
"14","Type","42+62","","","","","","","#1"
"Peter Willis","Peter","Willis","1234 Full Rd","","Somecity","Somestate","12345","USA","1","+1-555-555-5555","","foo@foo.foo","someusername1","h873hf8w7f82g8rfd2wf",""
```

