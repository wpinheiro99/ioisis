# IOISIS - I/O tools for converting ISIS data in Python

This is a Python library with command line interface
intended to access data from ISIS database files
and convert file formats.


## Command Line Interface (CLI)

To use the CLI command, use `ioisis` or `python -m ioisis`.
Examples:

```bash
# Read file.mst (and file.xrf) to a JSONL in the standard output stream
ioisis mst2jsonl file.mst

# Convert file.iso to an ASCII file.jsonl
ioisis iso2jsonl --jenc ascii file.iso file.jsonl

# Convert file.jsonl to file.iso where the JSON lines are like
# {"tag": ["field", ...], ...}
ioisis jsonl2iso file.jsonl file.iso

# Indirectly, convert file.mst to file.iso using jq
ioisis mst2jsonl file.mst \
| jq -c 'del(.active) | del(.mfn)' \
| ioisis jsonl2iso - file.iso
```

By default, the input and output are the standard streams,
but for MST+XRF input, where the MST must be given
and the matching XRF will be found based on the file name.

Try `ioisis --help` for more information.


## Library

To load ISIS data, you can use the `iter_records` function
of the respective module:

```python
from ioisis import iso, mst

# For MST files, you must use the filename
for record_dict in mst.iter_records("file.mst"):
    ...

# For ISO files, you can either use a file name
# or any file-like object open in "rb" mode
with open("file.iso", "rb") as raw_iso_file:
    for record_dict in iso.iter_records(raw_iso_file):
        ...
```

One can generate a single ISO record from a dict of data:

```python
>>> from ioisis import iso
>>> iso.dict2bytes({"1": ["testing"], "8": ["it"]})
b'000610000000000490004500001000800000008000300008#testing#it##\n'

```


### ISO construct containers (lower level data access Python API)

The `iso` module
uses the [Construct](https://github.com/construct/construct) library,
which makes it possible to create
a declarative "structure" object
that can perform bidirectional building/parsing
of bytestrings (instances of `bytes`)
or streams (files open in the `"rb"` mode)
from/to construct containers (dictionaries).


#### Building and parsing a single record

This low level data access
doesn't perform any string encoding/decoding,
so every *value* in the input dictionary
used for building some ISO data
should be a raw bytestring.
Likewise, the parser doesn't decode the encoded strings
(tags, fields and metadata),
keeping bytestrings in the result.

Here's an example
with a record in the "minimal" format expected by the ISO builder.
The values are bytestrings,
and each directory entry matches its field value based on their index.

```python
>>> lowlevel_dict = {
...     "dir": [{"tag": b"001"}, {"tag": b"555"}],
...     "fields": [b"a", b"test"],
... }

# Build a single ISO record bytestring from a construct.Container/dict
>>> iso_data = iso.DEFAULT_RECORD_STRUCT.build(lowlevel_dict)
>>> iso_data
b'000570000000000490004500001000200000555000500002#a#test##\n'

# Parse a single ISO record bytestring to a construct.Container
>>> con = iso.DEFAULT_RECORD_STRUCT.parse(iso_data)

# The construct.Container instance inherits from dict.
# The directory and fields are instances of construct.ListContainer,
# a class that inherits from list.
>>> [directory["tag"] for directory in con["dir"]]
[b'001', b'555']
>>> con.fields  # Its items can be accessed as attributes
ListContainer([b'a', b'test'])
>>> len(con.fields) == con.num_fields == 2  # A computed attribute
True

# This function directly converts that construct.Container object
# to a dictionary of already decoded strings in the the more common
# {tag: [field, ...], ..} format (default ISO encoding is cp1252):
>>> iso.con2dict(con).items()  # It's a defaultdict(list)
dict_items([('1', ['a']), ('555', ['test'])])

```


#### Other record fields

Each ISO record is divided in 3 parts:

* Label (24 bytes header with metadata)
* Directory (metadata for each field value, mainly its 3-bytes *tag*)
* Fields (the field values themselves as bytestrings)

The *label* has:

* Single character metadata (`status`, `type`, `coding`)
* Two numeric metadata (`indicator_count` and `identifier_len`),
  which should range only from 0 to 9
* Free room for "vendor-specific" stuff as bytestrings:
  `reserved`, `custom_2` and `custom_3`,
  where the numbers are their size in bytes
  and `reserved` has only one byte
* An entry map, i.e., the size of each field of the directory:
  `len_len`, `pos_len` and `custom_len`,
  which should range only from 0 to 9

```python
>>> con.len_len, con.pos_len, con.custom_len
(4, 5, 0)

```

Actually, the `reserved` is part of the entry map,
but it has no specific meaning there,
and it doesn't need to be a number.
Apart from the entry map and the not included length/address fields,
none of these metadata has any meaning when reading the ISO content,
and they're all filled with zeros by default
(the ASCII zero when they're strings).

```python
>>> con.status, con.type, con.coding, con.indicator_count
(b'0', b'0', b'0', 0)

```

Length and position fields that are stored in the record
(`total_len`, `base_addr`, `dir.len`, `dir.pos`)
are computed in build time and checked on parsing
(apart from `total_len`, which is ignored on parsing).
We don't need to worry about these fields,
but we can read them if needed.
For example, one directory record (a dictionary) has this:

```python
>>> con.dir[1]
Container(tag=b'555', len=5, pos=2, custom=b'')

```

As the default `dir.custom` field has zero length,
it's not really useful for most use cases.
Given that, we've already seen all the fields there are
in the low level ISO representation of a single record.


#### Tweaking the field lengths

The ISO2709 specification tells us
that a directory entry should have exactly 12 bytes,
which means that `len_len + pos_len + custom_len` should be 9.
However, that's not an actual restriction for this library,
so we don't need to worry about that,
as long as the entry map have the correct information.

Let's customize the length to get a smaller ISO
with some data in the `custom` field of the directory,
using a 8 bytes directory:

```python
>>> dir8_dict = {
...     "len_len": 1,
...     "pos_len": 3,
...     "custom_len": 1,
...     "dir": [{"tag": b"001", "custom": b"X"}, {"tag": b"555"}],
...     "fields": [b"a", b"test"],
... }
>>> dir8_iso = iso.DEFAULT_RECORD_STRUCT.build(dir8_dict)
>>> dir8_iso
b'0004900000000004100013100012000X55550020#a#test##\n'
>>> dir8_con = iso.DEFAULT_RECORD_STRUCT.parse(dir8_iso)
>>> dir8_con.dir[0]
Container(tag=b'001', len=2, pos=0, custom=b'X')
>>> dir8_con.dir[1]  # The default is always zero!
Container(tag=b'555', len=5, pos=2, custom=b'0')
>>> dir8_con.len_len, dir8_con.pos_len, dir8_con.custom_len
(1, 3, 1)

```

What happens if we try to build from a dictionary
that doesn't fit with the given sizes?

```python
>>> invalid_dict = {
...     "len_len": 1,
...     "pos_len": 9,
...     "dir": [{"tag": b"555"}],
...     "fields": [b"a string with more than 9 characters"],
... }
>>> iso.DEFAULT_RECORD_STRUCT.build(invalid_dict)
Traceback (most recent call last):
  ...
construct.core.StreamError: bytes object of wrong length, expected 1, found 2

```


### ISO files, line breaking and delimiters

The ISO files usually have more than a single record.
However, these files are created by simply concatenating ISO records.
That simple: concatenating two ISO files
should result in another valid ISO file
with all the records from both.

On the other hand,
there's an issue when it comes to line breaking in ISO files.
Although that's not part of the ISO2709 specification,
the `iso.DEFAULT_RECORD_STRUCT` parser/builder object
assumes that:

* All CR/LF (`\x0d` and `\x0a`, Carriage Return and Line Feed)
  characters must be ignored on parsing;
* All lines of a given record must have at most 80 bytes,
  and a line feed (`\x0a`) must be included after that;
* Every line must belong to a single record;
* The last line of a file must be `\x0a`.


#### Parsing/building data with meaningful line breaking characters

Suppose we want to store these values:

```python
>>> newline_info_dict = {
...     "dir": [{"tag": b"SIZ"}, {"tag": b"SIZ"}, {"tag": b"SIZ"}],
...     "fields": [b"linux^c\n^s1", b"win^c\r\n^s2", b"mac^c\r^s1"],
... }

```

That's makes sense as an example of an ISO record
with three `SIZ` fields, each with three subfields,
where the second subfield
is the default newline character of some environment,
and the third subfield is its size.
We can't build that using the `DEFAULT_RECORD_STRUCT`
because we know beforehand that our values have
these newline characters.
The alternative is to create another struct:

```python
>>> breakless_struct = iso.create_record_struct()
>>> newline_info_iso = breakless_struct.build(newline_info_dict)
>>> newline_info_iso
b'000950000000000610004500SIZ001200000SIZ001100012SIZ001000023#linux^c\n^s1#win^c\r\n^s2#mac^c\r^s1##'
>>> newline_info_con = breakless_struct.parse(newline_info_iso)
>>> newline_info_simple_dict = dict(iso.con2dict(newline_info_con))
>>> newline_info_simple_dict
{'SIZ': ['linux^c\n^s1', 'win^c\r\n^s2', 'mac^c\r^s1']}
>>> newline_info_iso == iso.dict2bytes(
...     newline_info_simple_dict,
...     record_struct=breakless_struct,
... )
True

```


#### Parsing/building with a custom line breaking and delimiters

The default builder/parser for a single record
was created with:

```python
DEFAULT_RECORD_STRUCT = iso.line_split_restreamed(
    iso.create_record_struct(
      field_terminator=iso.DEFAULT_FIELD_TERMINATOR,
      record_terminator=iso.DEFAULT_RECORD_TERMINATOR,
    ),
    line_len=iso.DEFAULT_LINE_LEN,
    newline=iso.DEFAULT_NEWLINE,
)
```

We can create a custom object using other values.
To use it, we'll pass that object
as the `record_struct` keyword argument
when calling the functions.


```python
>>> simple_data = {
...     "OBJ": ["mouse", "keyboard"],
...     "INF": ["old"],
...     "SIZ": ["34"],
... }
>>> custom_struct = iso.line_split_restreamed(
...     iso.create_record_struct(
...       field_terminator=b";",
...       record_terminator=b"@",
...     ),
...     line_len=20,
...     newline=b"\n",
... )
>>> simple_data_iso = iso.dict2bytes(
...     simple_data,
...     record_struct=custom_struct,
... )
>>> print(simple_data_iso.decode("ascii"), end="")
00096000000000073000
4500OBJ000600000OBJ0
00900006INF000400015
SIZ000300019;mouse;k
eyboard;old;34;@
>>> simple_data_con = custom_struct.parse(simple_data_iso)
>>> simple_data == iso.con2dict(simple_data_con)
True

```